# Linux 基础要点（Rocky Linux 8.10）

> 来源：2026-11-15 整理自前期学习笔记（手机端记录），已按本机 Rocky 8.10 实践校对。
> 标「**实测**」的，是这台 VM（10.0.0.150 / rocky@cycys）上真实跑出来的结果。

---

## 一、文件与目录

### 1. `mv`：移动 + 改名二合一

判断规则：

- 源、目标在**同一个目录** → 改名
- 目标是**其他目录路径** → 移动（保留原名）
- 目标路径**末尾写了新文件名** → 移动 + 改名

例子：

```bash
mv old.txt new.txt        # 同目录文件改名
mv text0 mydata           # 同目录目录改名
mv a.txt text1/           # 移进 text1，保留原名
mv a.txt text1/newfile.txt  # 移动并改名
```

易错点：

- 目标名字**已存在且是目录** → 源会被移动进该目录内部，而不是改名
- **移动目录不需要 `-r`**（`cp` 复制目录才要 `-r`，`rm` 删除目录必须 `-r`）
- 常用选项：`-i` 覆盖前询问、`-f` 强制覆盖、`-n` 不覆盖已有文件

### 2. `cat`：查看纯文本文件

- 适用：文本配置、代码、日志等人类可读内容
- 不适用：目录（报错 `Is a directory`）、二进制/图片/压缩包（乱码）、设备文件（可能卡住）
- **实测**：`cat text0`（目录）→ `Is a directory`；`cat /bin`（软链指向目录）→ 同样报 `Is a directory`

### 3. `ls`：列出目录内容

- `ls -l` 看详细属性；**软链接行首字母是 `l`**
- **实测**：`ls -l /bin` → `lrwxrwxrwx ... /bin -> usr/bin`

### 4. `file`：识别文件类型（看本质，不看扩展名）

| 输出 | 含义 |
|---|---|
| `ASCII text` / `UTF-8 text` | 文本文件 |
| `directory` | 目录 |
| `symbolic link to '路径'` | 有效软链接 |
| `broken symbolic link to '路径'` | 悬空失效软链接（目标被删除/移动） |
| `ELF 64-bit executable` | Linux 二进制可执行程序 |
| `gzip` / `Zip compressed data` | 压缩文件 |
| `PNG` / `JPEG image data` | 图片 |
| `block/character special`、`socket`、`FIFO` | 特殊文件 |
| `empty` | 空文件（大小 0） |

- **实测**：`file etc-full/mtab` → `broken symbolic link to ../proc/self/mounts`（相对软链换位置就失效）

### 5. `less` 与 `tail -f`

- `less 文件`：分页查看文本，`q` 退出；空格/`b` 翻页、`/关键词` 搜索
- `tail -f 文件`：实时跟踪日志刷新
- 本机没有 rsyslog：`/var/log/messages`、`/var/log/secure` **都不存在**，改用 `journalctl -f -u sshd`
- **实测**：`journalctl -f` 启动时会**先显示末尾 10 行**再跟随（别误以为是刚发生的）；退出用 `Ctrl+C`（`-f` 模式下 `q` 不一定生效）

### 6. 软链接（symbolic link）

- 类似 Windows 快捷方式：本身不存数据，只记录**目标路径**；目标消失就变 `broken`
- 相对路径的软链**换位置会失效**（副本里 6 条：`mtab`、`grub2.cfg`、`os-release`、`local-fs.target`、`motd.d/cockpit`、`issue.d/cockpit.issue`）
- Rocky 默认配色：青蓝色 = 软链接、深蓝色 = 目录、白色 = 普通文本、绿色 = 可执行文件/脚本

---

## 二、用户、权限、账号管理

### 1. `sudo`

- 作用：普通用户**临时**以 root 权限执行单条命令，输入**自己的**密码
- 前提：账号被授权（`/etc/sudoers`，编辑必须用 `visudo` 校验语法）
- 常用：`sudo 命令`、`sudo -i`（完整 root 登录环境）、`sudo -l`（看当前账号被允许执行什么）、`sudo -k`（清除密码缓存）
- Rocky/CentOS 默认：`wheel` 组拥有 sudo 权限，配置行 `%wheel ALL=(ALL) ALL`

### 2. `su`（switch user）

- `su root`：切换 root 但不加载完整环境（不推荐）
- `su - root`：**推荐**，模拟全新登录：完整环境变量 + 进入 root 家目录
- 需要输入**目标用户**的密码；适合整个终端切换身份
- **实测（11/13 踩坑）**：`su` 不带 `-` 时**只改 HOME/SHELL、不改当前目录**——cwd 仍在 `/home/rocky`，但 `~` 已变成 `/root`，导致 `cp -r /etc ~/text0` 写去 `/root/text0`。要完整切换用 `su -` 或 `sudo -i`；或一律写绝对路径

### 3. `usermod`（配 useradd / userdel 使用）

- 高频参数：
  - `-aG 组名 用户名`：**追加**附加组，保留原有组
  - `-G 组名`：**不加 `-a` 会覆盖**原有附加组 ⚠️
  - `-g` 改主组、`-s` 改登录 shell、`-l` 改用户名、`-L/-U` 锁/解锁、`-u` 改 UID
- 给普通用户加 sudo：`sudo usermod -aG wheel rocky`，**加完必须重新登录生效**
- 查看所属组：`id 用户名` / `groups 用户名`

### 4. 密码相关

- Linux **不保存明文密码**：加密后存 `/etc/shadow`，只能重置，不能查看原文
- 改密码：`passwd`（改自己）、`sudo passwd root`、`sudo passwd 其他用户`
- **忘记 root 密码**（无 sudo 时）：开机 GRUB 菜单按 `e` → `linux` 行末尾加 `rw init=/bin/bash` → `Ctrl+x` 启动单用户模式 → `passwd root` 重置 → 新版 SELinux 系统记得 `touch /.autorelabel`
- 备注：根分区只读挂载时，先 `mount -o remount,rw /` 再改密码

### 5. `exit`

- 直接输入 `exit` 退出当前 shell，回到上一个登录环境
- **不能写 `exit root`**（exit 不接收用户名参数）——**实测**：`bash: exit: root: 需要数字参数`
- 支持 `exit 数字` 指定退出状态码（0 = 正常退出，范围 0–255）

---

## 三、日志、时区、SSH 远程登录实验

### 1. 日志查看

- 传统 syslog 路径：`/var/log/secure`（新版 Rocky 默认**可能不存在**）
- systemd 统一日志：`journalctl -u sshd`（查 sshd 历史日志）；`journalctl -f -u sshd` 实时滚动
- 退出：`Ctrl+C`；`-f` 模式下 `q` 不一定生效
- **实测**：本机 `/var/log/messages`、`/var/log/secure` 都没有；`journalctl -u sshd` 能看到失败/成功登录记录和来源 IP

### 2. 时区 CST

- 当前场景：CST = **China Standard Time 中国标准时间 UTC+8**（北京时间）
- 注意 CST 多义：也可能指美国中部时间（UTC-6）等；国内服务器默认按北京时间理解
- 修改时区：`timedatectl set-timezone Asia/Shanghai`；查看：`timedatectl`

### 3. SSH 登录完整踩坑案例（实验 IP 10.0.0.150）

- `ssh cys@10.0.0.150` → `Permission denied` / `user unknown`：系统里**没有 cys 用户**
- 换成存在用户 `ssh rocky@10.0.0.150`，输密码 → 成功
- **关键判断**：能走到输密码这一步，说明网络、sshd 服务、防火墙、22 端口**全部正常**；失败几乎都是用户名不存在 / 密码错误
- 日志里：失败 = `authentication failure`、`user unknown`；成功 = `Accepted password for xxx`、`session opened`
- **实测（11/15）**：6 次 `Failed password for invalid user cys` + 1 次 `Accepted password for rocky`，来源都是 `10.0.0.150`（本机自发）
- 退出远程会话：输入 `exit`，连接正常关闭

---

## 四、和手账进度的对应

| 天 | 日期 | 用到本篇哪部分 |
|---|---|---|
| Day 2 | 11/13 | 一、文件与目录（cp/mv/rm）；二、su/sudo 环境问题；四、软链 |
| Day 3 | 11/14 | 一、find/which/file/cat/less/head |
| Day 4 | 11/15 | 三、journalctl 日志；一、file 软链；副本漂移排查 |
