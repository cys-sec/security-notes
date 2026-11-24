# Linux 14 天学习总结（Day 1–12 已完成）

> 汇总日期：2026-11-24　｜　数据来源：`git log`（远端 `96e8fd8`）、`Linux-14天计划打卡表.xlsx`、`03-notes/` 下 10 篇笔记
> 状态：**12 / 14 天完成**，命令手账 **36 / 60**（#1–#36）
> 本文件是索引与总览；每天的细节以各自的 `Linux-DayN-*.md` 为准。

---

## 一、总览表

| Day | 日期 | 主题 | 计划任务 | 实践反馈（关键发现 / 踩坑） | 笔记 |
|---|---|---|---|---|---|
| 1 | 11/12 | 文件与目录 | `pwd/ls/cd/cp/mv/rm` 各用一遍 | 建 `~/linux-practice/{src,dst}`，复制/改名/删除；**删前先 `ls`** | [Day1](Linux-Day1-文件与目录.md) |
| 2 | 11/13 | 文件与目录（练） | 复制 `/etc`、改名、删除 | 普通用户 `cp -r /etc` 只拷到 **87 项**；`su` 不带 `-` 使 `~` 指 `/root`；`cp` 目标不存在会"铺开"内容；最终 `sudo mv` 得 `etc-full`，`diff` 仅剩 6 条相对软链失效 | [Day2](Linux-Day2-复盘-2026-11-13.md) |
| 3 | 11/14 | 查找与查看 | `find/which/file/cat/less/head` | `which sshd`→`/usr/sbin/sshd`；`find -name` 必须加引号；`file mtab` 报 **broken symbolic link**（印证 Day2）；`services` 有 **11473 行**；非交互 `sudo` 需 `-t` 或 `-S` | [Day3](Linux-Day3-查找与查看.md) |
| 4 | 11/15 | 查找与查看（练） | 找 `.conf`、用 `less` 看日志 | 副本**过期**：副本 91 vs 真实 **112** 个 `.conf`；`rpm -q --last` 定因为 11/14 装了 open-vm-tools/fuse/fontconfig；`sudo cp -a /etc/.` 后对齐 112；`/var/log/messages` 不存在 → 改用 `journalctl` | [Day4](Linux-Day4-复盘-2026-11-15.md) |
| 5 | 11/16 | 文本处理 | `grep/sed/awk/cut` | 重点 `awk` 按列；用 `split` 统计协议；发现 **journald 默认不持久**（`/run/log/journal`，重启即丢） | [Day5](Linux-Day5-文本处理.md) |
| 6 | 11/17 | 文本处理（练） | 日志统计练习 | 完成"**爆破源 Top5**"：`journalctl` + `awk` + `sort/uniq`；Top1 仅 **3 次**、来源 `10.0.0.150`（自己）；60 分钟练习验证 | [Day6](Linux-Day6-文本处理（练）.md) · [任务2](../02-security/任务2-爆破源Top5.md) |
| 7 | 11/18 | 权限与用户 | `umask/chmod/chown/useradd/usermod/passwd` | `umask 0002` → 新建文件 664、目录 775；`chown` 必须 sudo；`useradd -m` + `chpasswd`；`usermod -aG wheel` 后 `id` 出现 `10(wheel)`、`sudo -l -U` 得 `(ALL) ALL`；`userdel -r` 清理 | [Day7](Linux-Day7-权限与用户.md) |
| 8 | 11/19 | 权限（练） | 验证组变更是否需要重登录 | **旧会话里 `id`/`cat`/`sudo` 全部无效，重登录后全部生效**（三样都验证了）；`test8` 用户实验后清理 | [Day8](Linux-Day8-权限（练）.md) |
| 9 | 11/20 | 进程与系统 | `ps/top/kill/systemctl/journalctl/uptime` | `ps aux --sort` 找高 CPU；发现"循环 `sleep 1`"其实是 **FinalShell 面板每秒轮询**（别 kill）；`systemctl --failed` 查出 `mcelog.service` 失败（AMD family 23 不支持，预期内） | [Day9](Linux-Day9-进程与系统.md) |
| 10 | 11/21 | 登录日志分析 | `journalctl -u sshd`、`last/lastb/w/lastlog` | 当天 `Accepted=1 / Failed=0`；11/19 起成功登录全是 `rocky` 来自 `10.0.0.1`（宿主机）；用 `$(NF-5)`/`$(NF-3)` 取用户名与来源 IP | [Day10](Linux-Day10-登录日志分析.md) |
| 11 | 11/22 | 网络 | `ip/ss/ping/traceroute/dig/curl` | `ens160=10.0.0.150/24`、网关/DNS `10.0.0.2`、宿主机 `10.0.0.1`；`ip neigh` 证实 `10.0.0.1`=VMnet8 主机网卡、`10.0.0.2`=NAT 网关；区分 curl **超时 vs 拒绝** | [Day11](Linux-Day11-网络.md) |
| 12 | 11/23 | 网络（练） | 端口基线 + 接口探测 | `sudo ss -tlnp` 只有 **22（sshd）**，`grep -v ':22'` 无输出 = 基线干净；UDP 68→67 是 DHCP；`curl` 三态：https **200/2.13s**、22 端口返回**协议错误(56)**、9 端口…；网关以 `ip route` 的 `default via` 为准 | [Day12](Linux-Day12-网络（练）.md) |
| 13 | — | 包管理+磁盘 | `dnf/rpm/df/du/free/lsblk/mount` | **未开始** | — |
| 14 | — | 压缩+日志安全 | `tar/scp/rsync/ssh-keygen/tail -f/lsof/last` | **未开始** | — |

---

## 二、跨天反复出现的 6 类经验

### 1. 权限与身份（Day2/4/7/8 反复出现）

- `/etc` 里有一部分文件**普通用户根本读不到**：同一目录 `find -type f`，普通用户 **399**、root **430**，差 31 个（`/etc/shadow`、`/etc/pki/private/*` 等）——所以 `cp -r /etc` 必然大量 `Permission denied`，**不是配置坏了**
- `sudo` 要输**自己的密码**；`su` / polkit 要输 **root 密码**
- **改用户/组 → 必须重新登录**才生效（Day8 用"旧会话 vs 新会话"三样对照证明了）
- 非交互环境（`ssh host "命令"`、脚本）里 `sudo` 报"需要终端" → `ssh -t` 或 `sudo -S`

### 2. 副本/快照会过期（Day4）

`etc-full` 是 11/13 22:52 的快照；11/14 装了三个包后 `/etc` 多了 21 个 `.conf`。**用 `rpm -q --last` 把"时间差"证明出来**，再 `cp -a` 同步。
> 结论：备份要标时间戳；"对不上"先查源端有没有变。

### 3. 路径语义（Day2/3/4）

- `cp -r 源 目标`：**目标不存在** → 内容铺进目标；**目标存在且是目录** → 生成 `目标/源名`
- `su` 不带 `-`：CWD 不变、但 `HOME` 变 → `~/xxx` 会指到 `/root/xxx`
- 不带 `/` 或 `./` 的词会被当**命令**执行（`etc-full` → "未找到命令"）

### 4. 日志去向变了（Day4/5/9/10）

RHEL8 没有 `/var/log/messages`、`/var/log/secure`；日志在 **journald**，且默认**不持久**（`/run/log/journal`，重启丢失）。查日志统一先 `journalctl`。

### 5. 学会识别"正常噪声"（Day9）

`ps` 里那个每秒 `sleep 1` 是 **FinalShell 面板轮询**；`systemctl --failed` 里的 `mcelog` 是硬件不支持——**不是故障，别乱 kill/乱改**。

### 6. 网络问题按层排查（Day11/12）

`ip neigh`（二层通不通）→ `ss`（端口在不在听）→ `curl`（协议/应用通不通）→ `dig`（域名解析）。顺序错了会白折腾。

---

## 三、缺口与待补

| 项 | 情况 | 建议 |
|---|---|---|
| ~~Day 1 / 3 / 5 / 6 无独立笔记~~ | **已补齐（2026-11-24）**：Day1/3/5/6 四篇已按当时真实输出还原 | — |
| 主机名不一致 | 早期笔记写 `rocky@rocky` 或 `rocky@cycs`，实际已是 `cys` | 统一为 `cys` |
| Day 13 / 14 | 未开始 | 按计划继续：包管理与磁盘、压缩与日志安全 |

---

## 四、数据与证据

- 打卡表：`Linux-14天计划打卡表.xlsx` → `01-14天计划` 第 1–12 天 = 完成；`02-60命令打卡` #1–#36 = 完成
- 远端仓库：`cys-sec/security-notes`，`main` = `96e8fd8`
- 笔记清单（`03-notes/`）：Day2、Day4、Day7、Day8、Day9、Day10、Day11、Day12 + 两篇 GitHub 专题

---

## 五、下一步

1. Day13 包管理与磁盘：`dnf/rpm/df/du/free/lsblk/mount`（用 `dnf provides` 查文件属于哪个包）
2. Day14 压缩与日志安全：`tar/scp/rsync/ssh-keygen/tail -f/lsof/last`（把前 12 天的日志分析串成一次完整排查）
3. 补 Day1/3/5/6 四篇笔记后，本目录就是一套完整的 14 天记录
