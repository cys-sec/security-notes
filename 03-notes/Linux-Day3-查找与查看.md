# Linux Day 3：查找与查看（2026-11-14）

> 环境：Rocky Linux 8.10，用户 `rocky`；素材用 Day2 的完整副本 `~/text0/etc-full`（429 个文件）。
> 来源：`01-linux/命令手账.md` 的「day3 实操」段（当时真实终端输出）。

---

## 一、今天的目标

学会"**不认识的东西先问工具**"：找文件用 `find`，找命令用 `which`，判类型用 `file`，看内容分长短用 `cat/head/less`。

## 二、命令与实测

### 1. `find` —— 按条件找文件

```bash
$ find ~/text0/etc-full -name "*.conf" -type f | wc -l
87
$ find ~/text0/etc-full -name "*.conf" -type f | head -3
/home/rocky/text0/etc-full/gcrypt/random.conf
/home/rocky/text0/etc-full/dbus-1/session.conf
/home/rocky/text0/etc-full/dbus-1/system.conf
$ find /etc -maxdepth 1 -type f -name "*conf*" | head -5
/etc/resolv.conf
/etc/sudo.conf
/etc/libuser.conf
```

- **`find`**：语法 `find 路径 [条件] [动作]`；作用：按名字/类型/大小/时间递归查找。常用 `-name`（名字，**必须加引号**）、`-type f|d|l`（文件/目录/软链）、`-size +100k`、`-mtime -7`、`-maxdepth N`

### 2. `which` —— 命令装在哪

```bash
$ which bash
/usr/bin/bash
$ which sshd
/usr/sbin/sshd
$ echo $PATH
/home/rocky/.local/bin:/home/rocky/bin:/usr/local/bin:/usr/bin:/usr/local/sbin:/usr/sbin
```

- **`which`**：语法 `which 命令名`；作用：在 `PATH` 里找可执行文件位置。找不到时用 `command -v` 或直接写绝对路径（`/usr/sbin/sshd`）

### 3. `file` —— 判断文件本质

```bash
$ file /bin/ls
/bin/ls: ELF 64-bit LSB shared object, x86-64, dynamically linked, ... stripped
$ file ~/text0/etc-full/services
.../services: UTF-8 Unicode text
$ file ~/text0/etc-full/mtab
.../mtab: broken symbolic link to ../proc/self/mounts
$ file ~/text0/etc-full/passwd
.../passwd: ASCII text
```

- **`file`**：语法 `file 文件`；作用：看文件真实类型（不看扩展名）；**软链会直接报目标在不在**（`broken symbolic link`）

### 4. `cat` / `head` / `less` —— 看内容，按长短选工具

```bash
$ cat /etc/hostname
cys
$ wc -l /etc/hostname
1 /etc/hostname

$ wc -l ~/text0/etc-full/services
11473 ~/text0/etc-full/services
$ head -n 5 ~/text0/etc-full/services
# /etc/services:
# $Id: services,v 1.49 2017/08/18 12:43:23 ovasik Exp $
#
# Network services, Internet style
# IANA services version: last updated 2016-07-08

$ less -F /etc/hostname
cys
```

- **`cat`**：语法 `cat 文件`；作用：一次输出全部内容，**只适合短文件**
- **`head`**：语法 `head -n N 文件`；作用：只看前 N 行（大文件先看头部）
- **`less`**：语法 `less 文件`；作用：交互式分页；`q` 退出、空格/`b` 翻页、`/关键词` 搜索、`G` 末尾、`gg` 开头

## 三、实践反馈

| 观察 | 结论 |
|---|---|
| `find ... -name "*.conf"` | 通配符**必须加引号**，否则 shell 先展开、find 收不到模式 |
| `file mtab` 报 broken symlink | 直接印证 Day2 `diff` 里那 6 条**相对软链接**失效，不是文件丢失 |
| `services` 有 11473 行 | 这就是"别用 `cat` 怼大文件"的实证 |
| `sudo which sshd` 在 `ssh host "命令"` 里失败 | 非交互环境 `sudo` 报"需要终端"→ 用 `ssh -t` 分配终端，或 `sudo -S` 从标准输入读密码 |
| `which sshd` 能找到 | 本机 `PATH` 含 `/usr/sbin`（不是所有发行版默认都含） |

## 四、和前后几天的联系

- 前一天（Day2）证明"副本会缺东西"，今天用 `find` + `file` 做**验证工具**
- 后一天（Day4）用 `comm` 比对"副本 vs 真实 `/etc`"，正是 `find` 输出的延伸
