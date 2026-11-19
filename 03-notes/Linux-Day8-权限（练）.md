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
