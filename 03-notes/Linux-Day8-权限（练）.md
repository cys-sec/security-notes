# Linux Day 8：权限（练）——组变更要重登录（2026-11-19）

> 环境：Rocky 8.10（rocky@rocky）；实验用户 `test8`（实验后清理）。
> 证据：用户终端截图（2026-11-19 21:31–22:07）。

---

## 一、实验目标

亲手验证「组变更要重新登录才生效」，并分别观察三样东西在**旧会话**与**重登录后**的差异：

1. `id` 显示的附加组
2. 按组授权的**文件访问**（root:wheel 640 的文件）
3. `sudo` 是否放行（wheel 组授权）

---

## 二、过程与结果

### 1. 基线（test8 未加组）

```text
[test8@rocky ~]$ id
uid=1002(test8) gid=1002(test8) groups=1002(test8) context=unconfined_u:...

[test8@rocky ~]$ sudo ls /root
[sudo] password for test8:
test8 is not in the sudoers file. This incident will be reported.
```

> 不在 wheel → sudo 先验密码、再查 `/etc/sudoers`，最后报"不在 sudoers 文件中"。

### 2. 加组（在 rocky 会话里执行）

```text
[rocky@rocky ~]$ sudo usermod -aG wheel test8
[rocky@rocky ~]$ id test8
uid=1002(test8) gid=1002(test8) 组=1002(test8),10(wheel)
```

> `-aG` 追加组成功；`id test8` 立刻显示 `10(wheel)`——它读的是 `/etc/group`。

### 3. 旧会话（加组后、未重登录）——三样全不变

```text
[test8@rocky ~]$ id
uid=1002(test8) gid=1002(test8) groups=1002(test8) ...        ← 仍无 wheel

[test8@rocky ~]$ cat /tmp/wheel-readable
cat: /tmp/wheel-readable: Permission denied                   ← 组权限不生效

[test8@rocky ~]$ sudo ls /root
[sudo] password for test8:
test8 is not in the sudoers file. ...                        ← sudo 也不放行
```

### 4. 重登录后——三样全变

```text
[test8@rocky ~]$ exit
logout
[test8@rocky ~]$ su - test8
Password:
[test8@rocky ~]$ id
uid=1002(test8) gid=1002(test8) groups=1002(test8),10(wheel) ...

[test8@rocky ~]$ cat /tmp/wheel-readable
（成功，无输出）

[test8@rocky ~]$ sudo ls /root
[sudo] password for test8:
anaconda-ks.cfg
```

---

## 三、结论

1. **组变更对已登录会话不生效**：`id`、按组授权的文件访问、`sudo` 授权，全部按登录时确定的组列表判断 → **必须重新登录**。
2. **`id 用户名` 不能当"已生效"的证据**：它查 `/etc/group` 记录，当然立刻显示 wheel；要验证生效，看的是**当前会话的 `id`**（无参数）。
3. **安全提醒**：test8 进 wheel 后拥有 root 能力——实验里它甚至能通过 `sudo userdel` 操作自己。实验结束必须删除用户或撤组。
4. **uid 会变**：这次 test8 从 `1001` 变成 `1002`，因为删了重建会重新分配；要固定 uid 得 `useradd -u`。

---

## 四、踩坑记录

| 现象 | 原因 | 正确做法 |
|---|---|---|
| `usermod: shell 组不存在` | 组名打错成 `shell` | Rocky 的 sudo 组是 `wheel`（`sudo grep wheel /etc/sudoers`） |
| `usermod: 用户 test8 不存在` | 用户已被删除 | 先 `sudo useradd -m test8` 重建 |
| `sudo -l` / `usermod` 在 test8 会话里执行报 sudoers 错误 | `usermod` 需要 root，test8 无 sudo | 切到 rocky 会话（或 `sudo su -`）执行 |
| `userdel: user test8 is currently used by process …` | 在 test8 **自己的会话**里删自己 | 先退出 test8 会话 / `sudo pkill -u test8`，再在 rocky 会话删 |
| 密码 `Test@1234` 被当命令敲 | 误输入 | 密码不要出现在命令行（用 `chpasswd` 管道或交互式 `passwd`），并清理 `~/.bash_history` |

---

## 五、清理步骤（实验结束）

```bash
# 在 rocky 会话中执行
who                        # 确认没有残留的 test8 会话
sudo pkill -u test8        # 结束 test8 的所有进程（若有）
sudo userdel -r test8      # 删用户 + 家目录
id test8                   # 期望：无此用户
ls /home/test8             # 期望：不存在
sudo rm -f /tmp/wheel-readable
```

> 若仍报"进程占用"，先退出所有 test8 的 shell，再执行；`userdel -f -r` 可强制（但会跳过检查，慎用）。

**清理结果（2026-11-19 22:44 实测）**：`sudo pkill -u test8` 后 `ps -u test8` 只剩表头；`sudo userdel -r test8` 成功；`id test8` → `无此用户`；`ls -d /home/test8` → 不存在；`/tmp/wheel-readable` 已删除。实验环境干净。

---

## 六、附：`ps` / `pkill` 小结（清理实验用到的）

> 来源：手机端整理（2026-11-19）；正好与 Day 9「进程与系统」衔接。

### `ps`：进程快照

- `ps`：当前终端的进程快照
- `ps -u 用户名`：只列出属主为该用户的进程
- 表头字段：

| 字段 | 含义 |
|---|---|
| PID | 进程唯一编号，`kill` 杀进程时用 |
| TTY | 终端：`pts` 是虚拟终端，`?` 表示没有终端 |
| TIME | 进程**累计占用 CPU 的时间**，不是墙钟运行时长 |
| CMD | 启动该进程的命令 |

- 输出为空 = 该用户没有进程在跑
- 想看"墙上时间"对比 CPU 时间，用 `ps -o pid,etime,time,cmd -u 用户名`（`etime` 是已运行时长，`time` 是累计 CPU）

### `pkill`：按条件批量结束进程

- `pkill -u 用户名`：结束该用户的所有进程（默认发 `SIGTERM`，进程可自行清理并退出）
- 进程不响应时：`pkill -9 -u 用户名`（`SIGKILL`，强杀，进程无法拦截）
- 实验用途：清掉 test8 的登录 shell/会话，释放心资源，为 `userdel -r` 铺路

### 配套流程（本次实测）

1. `sudo pkill -u test8` —— 清进程
2. `sleep 1` —— 等进程真正退出
3. `ps -u test8` —— 确认无进程（只剩表头）
4. `sudo userdel -r test8` —— 删除用户
5. 若有残留进程，`userdel` 会报 `user ... is currently used by process ...`，删除失败

补充：`ps -u` 看的是**进程归属**，不依赖 utmp，所以能查到 `su` 出来的 shell；`who` 只读 utmp，看不到这类会话。

---

## 七、命令语法与作用（Day 8 用到的）

- **`su`**：语法 `su [-] [用户]`；作用：切换用户，`-` 表示模拟完整登录环境（工作目录与环境变量都切换）
- **`id`**：语法 `id [用户]`；作用：显示 uid/gid 与附加组；不带参数看当前会话，带用户名看账户记录
- **`sudo`**：语法 `sudo [-u 用户] 命令`；作用：以 root（或其他用户）身份执行单条命令，按 sudoers 规则授权
- **`usermod`**：语法 `usermod -aG 组 用户`；作用：修改用户附加组（`-a` 追加，漏掉会覆盖原有组）
- **`userdel`**：语法 `userdel [-r] 用户`；作用：删除用户，`-r` 连家目录；该用户有进程时会失败
- **`ps`**：语法 `ps [选项]`；作用：进程快照；`-u 用户` 按属主过滤、`-o 字段列表` 自选列
- **`pkill`**：语法 `pkill [-信号] [-u 用户] [模式]`；作用：按条件批量结束进程（默认 SIGTERM）
- **`cat`**：语法 `cat [文件]`；作用：输出文件内容（本课用来验证文件组权限是否生效）
