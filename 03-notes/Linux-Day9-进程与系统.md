# Linux Day 9：进程与系统（2026-11-20）

> 环境：Rocky 8.10（rocky@rocky）；证据：用户终端截图（2026-11-20 22:07–22:52）。
> 素材：sshd 服务、FinalShell 监控轮询、一次 `sleep 300 &` 的 kill 实验。

---

## 一、ps —— 进程快照

```bash
ps -ef | head -5
ps aux | sort -k3 -nr | head -5
ps -u rocky
```

- **语法**：`ps [选项]`；常用 `-ef`（全量 + PPID）、`aux`（带 CPU/内存）、`-u 用户`、`-eo 字段列表`、`--sort=-%cpu`
- **作用**：查看当前进程的一次性快照（不刷新），排查"谁在跑、谁在吃资源"

观察：

- `ps -ef` 前几行是 PID 1（systemd）和内核线程（`[kthreadd]`、`[rcu_gp]` 等，方括号表示内核线程）
- `ps aux | sort -k3 -nr` 里排前面的是自己会话的 `sshd`、`tuned` 等
- **发现一个"循环 sleep"**：`pgrep -a -u rocky sleep` 显示 `sleep 1`，用 `ps -eo pid,ppid,etime,cmd --forest` 定位到父进程是：

  ```
  bash -c 'export LANG="en_US";...;free;echo finalShell_separator;uptime;...;sleep 1;...'
  ```

  这是 **FinalShell 的系统信息面板每秒轮询**（`finalShell_separator` 是它的输出分隔标记），不是系统异常，也不要去 kill 它。

---

## 二、top —— 实时进程与负载

```bash
top
```

- **语法**：`top [选项]`；交互键 `P` 按 CPU 排序、`M` 按内存排序、`1` 看每核、`q` 退出；`-b -n 1` 出一帧快照
- **作用**：动态查看进程与系统负载（1/5/15 分钟 load average），定位高占用进程

实测读数：`Tasks: 211 total, 1 running, 210 sleeping`、`load average: 0.15, 0.12, 0.09`、`Mem: 486M / 3.5G`、`Swap: 0` —— 练习机基本空转，正常。

---

## 三、kill —— 向进程发送信号

```bash
sleep 300 &
jobs -l
kill %1
jobs -l          # 期望 [1]+ 已终止 / Terminated
```

- **语法**：`kill [-信号] 目标`；目标为 PID 或 `%作业号`；默认 `SIGTERM(15)`，`-9` = `SIGKILL`
- **作用**：向进程发送信号；15 让进程清理后退出，9 由内核强制终止（进程无法拦截）

实测：

- `sleep 300 &` → `[1] 174933`；`jobs -l` → `[1]+ 174933 运行中`；`kill %1` 后 `jobs -l` → `[1]+ 174933 Terminated` ✅
- 坑 1：`kill -9 <PID>` 直接报语法错误 —— `<PID>` 是**占位符**，`<` 被 shell 当成输入重定向，必须换成真实数字
- 坑 2：`kill 197944` 报 `No such process` —— 进程已自行退出，kill 不存在的 PID 就是报错
- 坑 3：`echo $?` 在管道后取的是 **grep** 的退出码（找到匹配=0），不是 kill 的
- 坑 4：`ps | grep sleep` 里的 `sleep 1` 是 FinalShell 轮询，PID 每次不同，不是"杀不死的同一个进程"

---

## 四、systemctl —— 管理 systemd 服务

```bash
systemctl status sshd --no-pager
systemctl --failed
```

- **语法**：`systemctl [选项] 子命令 [单元]`；常用 `status/start/stop/restart/enable/disable/is-active/is-enabled/list-units/--failed`；`--no-pager` 不分页
- **作用**：查服务状态、启停、开机自启、列失败单元

实测：

```
● sshd.service - OpenSSH server daemon
     Loaded: loaded (/usr/lib/systemd/system/sshd.service; enabled; vendor preset: enabled)
     Active: active (running) since Fri 2026-11-20 21:18:58 CST; 1h 20min ago
   Main PID: 1004 (sshd)
      Tasks: 1 (limit: 23005)
     Memory: 3.7M
```

- `systemctl --failed` 命中 1 个：**`acelog.service`（Machine Check Exception Logging Daemon）**，状态 `loaded failed failed`
- 该服务在 VMware 虚拟机里失败是常见现象（没有真实 MCE 硬件/对应设备），属于"可以确认无害的失败"——但要先看原因再下结论：`systemctl status acelog --no-pager`、`journalctl -u acelog -n 20 --no-pager`

---

## 五、journalctl —— 查询 systemd 日志

```bash
journalctl -u sshd -n 20 --no-pager
journalctl -p err -b --no-pager | head
journalctl --disk-usage
```

- **语法**：`journalctl [选项] [匹配模式]`；常用 `-u 单元`、`-n 行数`、`-b`、`-p 级别`、`-f`、`--since/--until`、`-o 格式`、`--disk-usage`
- **作用**：查 journald 日志，按服务/时间/级别过滤；`--disk-usage` 看占用

实测：

- `journalctl -u sshd -n 20` 里能看到 **11/19 的登录记录** 和多次 `-- Reboot --` 分隔——说明 Day 5 打开的日志持久化一直在生效（重启不丢）
- 日志里还能看到主机名从 `cys` 变成 `rocky` 的历史（`11月 19 21:03:36 cys` → `11月 19 21:47:30 rocky`）
- `journalctl -p err -b | head` 只有 1 条：`kernel: pcie4 msi0000:00:07.3: SMBus Host Controller not enabled!` —— VMware 虚拟机常见的无害告警
- `journalctl --disk-usage` → **96.0M**（Day 5 当时是 16.0M，因为持久化 + 多次重启后日志累积）

---

## 六、uptime —— 运行时长与负载

```bash
uptime
```

- **语法**：`uptime`（`-p` 输出"已运行多久"）
- **作用**：显示当前时间、开机时长、登录用户数、1/5/15 分钟平均负载

实测：`22:07:02 up 48 min, 3 users, load average: 0.27, 0.17, 0.11`。

---

## 七、命令语法与作用（汇总）

| 命令 | 语法 | 作用 |
|---|---|---|
| `ps` | `ps [选项]`（`-ef`/`aux`/`-u 用户`/`-eo 字段`） | 进程快照 |
| `top` | `top [选项]`（交互键 P/M/1/q） | 实时进程与负载 |
| `kill` | `kill [-信号] PID` 或 `kill %作业号` | 向进程发信号（默认 TERM，-9 强杀） |
| `systemctl` | `systemctl [选项] 子命令 [单元]` | 管理 systemd 服务/单元 |
| `journalctl` | `journalctl [选项] [匹配]` | 查询 systemd 日志 |
| `uptime` | `uptime` | 运行时长与平均负载 |

---

## 八、结论

1. 六条命令全部跑通；系统状态健康（sshd running、负载很低、内存充足）。
2. `systemctl --failed` 命中的 `acelog.service` 与 `journalctl -p err` 里的 SMBus 告警，都是**虚拟机环境无害项**，但排查流程要记住：先看状态/日志再判断。
3. 日志持久化验证成功：跨重启的 sshd 记录可查（11/19 → 11/20）。
4. 排查工具组合：`ps --forest` 定位进程父子关系、`jobs -l` 看后台作业、`systemctl --failed` + `journalctl -p err -b` 做快速体检。
