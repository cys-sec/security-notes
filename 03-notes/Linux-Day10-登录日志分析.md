# Linux Day 10：登录日志分析（2026-11-21）

> 环境：Rocky 8.10（rocky@rocky）；证据：终端截图（2026-11-21 21:41–22:10）。
> 素材：本机 sshd 日志 + wtmp/btmp/lastlog + 当前会话。

---

## 一、今日登录概览

```bash
journalctl -u sshd --since today -o cat
journalctl -u sshd --since today -o cat | grep -c 'Accepted'
journalctl -u sshd --since today -o cat | grep -c 'Failed'
```

- **语法**：`journalctl -u 单元 --since 时间 -o cat`；`grep -c` 只输出匹配行数
- **作用**：先看 sshd 今天的原始消息（`-o cat` 去掉时间/主机前缀），再统计成功/失败次数

实测：`Accepted` = **1**、`Failed` = **0**；当天 sshd 启动日志为 `Server listening on 0.0.0.0 port 22` + `Server listening on :: port 22`。

---

## 二、成功登录明细

```bash
journalctl -u sshd --since today --no-pager | grep 'Accepted'
journalctl -u sshd --since 2026-11-19 --no-pager | grep 'Accepted' | awk '{print $(NF-5), $(NF-3)}'
```

- **语法**：`grep 'Accepted'` + `awk '{print $(NF-5), $(NF-3)}'`
- **作用**：列出成功登录；`NF-5` 是用户名、`NF-3` 是来源 IP（从末尾数，不受 journalctl 前缀影响）

实测：11/19 以来的成功登录全部是 **`rocky` 从 `10.0.0.1`**（自己的 Windows 宿主机），没有陌生来源。

---

## 三、登录历史与当前会话

```bash
last -n 10
lastb -n 10
who
w
lastlog
```

- **语法/作用**：
  - `last`：读 `/var/log/wtmp`，成功登录的历史（谁、从哪、何时、登入/登出）
  - `lastb`：读 `/var/log/btmp`，**失败**登录历史；需要 root（`/var/log/btmp` 权限 600）
  - `who`：当前登录会话（只读 utmp，**看不到 `su` 出来的会话**）
  - `w`：当前登录 + 正在执行什么 + 负载
  - `lastlog`：每个用户**最后一次登录**时间

实测：

- `last` 里当前有 3 个会话（两个 pts 来自 10.0.0.1、一个 tty1 控制台）
- `lastb`：需要 `sudo -i` 后才有权限；里面记录的都是**失败的用户名**：`test8`（11/19 的 su 失败）、`baduser`（11/17 的 ssh 失败）、`1233456`/`pkili`（11/20 23:18 控制台输错的用户名）
- `who` / `w`：3 个会话、`up 30 min`、load 0.35/0.23/0.13
- `lastlog`：root 最后登录 11/13、rocky 11/21、test8 11/19（**用户已删但 lastlog 记录残留**，正常现象）

---

## 四、异常与安全视角

```bash
journalctl -u sshd --since 2026-11-19 --no-pager | grep -E 'Failed|invalid' | awk '{print $(NF-3)}' | sort | uniq -c | sort -rn
journalctl -u sshd --since 2026-11-19 --no-pager | grep -oE 'invalid user [a-zA-Z0-9_-]+' | sort | uniq -c | sort -rn
```

- **语法**：`grep -E 'A|B'`（扩展正则的"或"）；`grep -oE` 只输出匹配到的片段
- **作用**：按来源 IP 统计失败尝试、按被撞用户名统计——爆破源 Top N 的标准做法

实测：两条**都没有输出**——因为 11/19 以来 sshd 没有 `Failed`/`invalid` 记录（11/17 那次 baduser 在 `--since` 范围之外）。**没有输出就是好消息**。

---

## 五、重要发现：`last` 里有 3 次 `crash`

```
reboot  system boot  4.18.0-553.134.1  Fri Nov 20 23:27 - crash (22:13)
reboot  system boot  4.18.0-553.134.1  Fri Nov 20 23:26 - crash (00:07)
...
rocky   tty1                            Fri Nov 20 23:15 - 23:19  (00:03)
```

- `crash` 表示这次关机**没有正常记录 shutdown**（wtmp 里没写正常的关机行）——通常是**强制断电、VM 强制关机/重置、或系统崩溃**，而不是 `shutdown/reboot` 的正常重启
- 时间点：11/20 23:26 一次（只活了 7 分钟）、11/20 23:27 一次（活了 22 小时 13 分，直到 11/21 21:40 左右）、以及今天 21:40 前后那次重启
- 排查方法：

  ```bash
  journalctl -b -1 -p err --no-pager | tail -30
  journalctl -b -2 -p err --no-pager | tail -30
  ```

  > **语法**：`journalctl -b -N` 查第 N 次开机（`-1` = 上一次）；**作用**：看崩溃前最后一次开机有没有错误日志（OOM、内核错误、I/O 错误）

- 还要考虑宿主机侧原因：VMware 是否休眠/挂起、是否手动 reset、磁盘/内存是否吃紧

---

## 六、命令语法与作用（汇总）

| 命令 | 语法 | 作用 |
|---|---|---|
| `journalctl` | `journalctl [选项] [匹配]` | 查 systemd 日志（`-u` 服务、`--since` 时间、`-b` 开机、`-p` 级别） |
| `last` | `last [-n 行数]` | 成功登录历史（wtmp） |
| `lastb` | `lastb [-n 行数]` | 失败登录历史（btmp，需 root） |
| `who` | `who` | 当前登录会话（utmp，不含 su 会话） |
| `w` | `w` | 当前会话 + 正在执行的命令 + 负载 |
| `lastlog` | `lastlog` | 每个用户最后一次登录 |

---

## 七、结论

1. 今天 sshd：**成功 1 次、失败 0 次**；11/19 以来所有成功登录都是 `rocky@10.0.0.1`（自己），无异常来源。
2. `lastb` 里的失败记录都是"自己输错账号"（test8/baduser/1233456/pkili），不是攻击。
3. **`last` 发现的 3 次 `crash` 需要留意**——说明有非正常关机；如果是为了改名/折腾而强制重启 VM，那就是原因；否则应查 `journalctl -b -1 -p err` 和 VMware 侧。
4. 用户名改名（rocky → cys）尚未完成：`who`/`lastlog`/提示符都还是 rocky；改名需要"rocky 无登录进程"时操作（控制台 root 或临时管理员）。

---

## 八、遗留

- [ ] 用户名改名 rocky → cys（需控制台/临时管理员方式，见 Day 9 后续讨论）
- [ ] 排查 11/20–11/21 的 crash 记录（`journalctl -b -1 -p err`）

---

## 九、手机端知识点汇总（2026-11-21 晚整理）

> 来源：手机端整理（覆盖 Day 9 进程 + Day 10 日志）；已与仓库笔记合并。带 ⚠️ 的是补充的精确说明。

### 1. 进程查看与管理

- `ps` 输出字段：PID/PPID、TTY（`pts` 虚拟终端、`?` 无终端）、TIME（累计 CPU 时间，不是墙钟）、CMD
  - **语法**：`ps [选项]`；`-eo 字段列表` 自定义列（如 `ps -eo pid,ppid,etime,cmd`）
- `top`：交互式监控，看 load average、CPU/内存、进程；快捷键 `P`（CPU 排序）、`M`（内存排序）、`1`（每核）、`q`（退出）
- `kill PID`（默认 SIGTERM）/ `kill -9`（SIGKILL 强杀）/ `kill %1`（**作业号**，只对当前 shell 的作业有效）/ `pkill -u 用户`（按用户批量结束）
- ⚠️ `load average` 的准确含义：**可运行 + 不可中断睡眠（D 状态）任务的平均数**（1/5/15 分钟指数衰减），不是单纯的"等待 CPU 的任务数"；单核 load 1.0 ≈ 满载。

### 2. 日志与 SSH 登录排查

- `journalctl -u sshd`：按服务过滤；`--since` 时间范围；`--no-pager` 不进分页；`-p` 按优先级过滤
- `grep` 组合：`-c` 统计行数、`-n` 带行号、`-l` 只输出有匹配的**文件名**（多文件场景才有意义）、`-E 'A|B'` 扩展正则"或"、`-oE` 只提取匹配片段
- `awk` 提取字段：`NF` 一行总字段数、`$NF` 最后一列；sshd `Accepted` 行用 `$(NF-5)` 取用户名、`$(NF-3)` 取来源 IP（**依赖消息格式固定**，换日志类型要重新数）
- `last`（读 wtmp，成功登录）/ `lastb`（读 btmp，失败登录，需 root）/ `lastlog`（各用户最后一次登录）
- `w` 对比 `who`/`last`：`w` 多显示"正在执行的命令 + 负载/空闲时间"
- ⚠️ `last` 里的 `crash` = 这次关机**没有正常记录 shutdown**（非正常断电/强制重置/崩溃），不是"某个程序崩溃"。

### 3. 用户、主机名与终端

- 提示符 `用户@主机`；root 提示符 `#`、普通用户 `$`
- 改用户名：**必须先清掉该用户的进程/会话**（否则 `usermod: user ... is currently used by process ...`），再配套改家目录、主组；从独立 root 会话（控制台或临时管理员）操作
- `exit`：root shell 里退回原普通用户；普通用户 exit 即关闭会话

### 4. 服务与权限

- `systemctl`：管理系统服务（`status`/`start`/`stop`/`restart`/`enable`/`disable`）
- `chown`：改文件属主/属组；`chmod -R`：递归改权限
- `ls -d`：看目录本身而不是目录内容（配合 `*/` 通配时尤其有用）

### 5. 星期缩写

`Mon`、`Tue`、`Wed`、`Thu`、`Fri`、`Sat`、`Sun`（`last`、`date` 等输出里常见）

### 6. 一句话串记

> `top/ps/w` 看进程在线；`journalctl + grep/awk` 抓 SSH 日志；`last/lastb` 查登录成败；改用户先清进程；`exit` 分层退出。
