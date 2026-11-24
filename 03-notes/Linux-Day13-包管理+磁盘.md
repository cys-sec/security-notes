# Linux Day 13：包管理 + 磁盘（2026-11-24）

> 环境：Rocky 8.10（rocky@rocky）；证据：终端输出（2026-11-24 19:31–20:20）。
> 结论：系统健康——根分区 12%、内存 available 3.0G、swap 未用、无大文件堆积。

---

## 一、rpm 查询（#40）

```bash
rpm -q openssh-server
rpm -qa | wc -l
rpm -qi openssh-server
rpm -ql openssh-server | head
rpm -qf /usr/sbin/sshd
```

- **语法**：`rpm -q 包名`（是否安装）；`-qa` 全部已装；`-qi` 信息；`-ql` 包内文件；`-qf 文件路径` 反查所属包
- **作用**：离线查"装了什么、来自哪个包、这个文件属于谁"

实测：

- `openssh-server-8.0p1-29.el8_10.x86_64` 已安装；`rpm -qa | wc -l` = **531** 个包
- `rpm -qi` 里能看到安装日期 `2026年10月28日 21:43:27`、大小 `1047503`、签名仓库等
- ⚠️ 坑：`rpm -q` 不带参数会报"未给出要查询的参数"；`rpm -qf /usr/sbin /sshd`（写成两个路径）会把 `/usr/sbin` 当包查询、`/sshd` 当文件——正确写法是 `rpm -qf /usr/sbin/sshd`

---

## 二、dnf 仓库与查询（#41 #42）

```bash
dnf repolist
dnf provides */ssh
dnf info tree
dnf history | head
```

- **语法**：`dnf 子命令 [关键字]`
- **作用**：`repolist` 看仓库；`provides` 按文件反查包；`info` 看包详情；`history` 看安装/卸载历史

实测：

- 仓库 3 个：`appstream`、`baseos`、`extras`
- `dnf provides */ssh` 第一次会**下载元数据**（几十 MB，慢，可 `^C` 中断，第二次已缓存）；命中很多包（bash-completion、cockpit-bridge、golang-src、openssh、openssh-clients、rear、rhel-system-roles、selinux-policy…）
- ⚠️ 教训：通配符 `*/ssh` 太宽，容易刷屏；查"某个命令属于哪个包"更推荐 `dnf provides /usr/bin/ssh` 或 `rpm -qf $(which ssh)`
- `dnf history` 显示 4 次事务：10/28 初始安装（462 包）、11/06 装 vim、11/14 装 open-vm-tools（69 包）、11/22 装 traceroute

---

## 三、dnf 安装 / 卸载 / 更新检查（#37 #38 #39）

```bash
sudo dnf install -y tree
tree --version
sudo dnf remove tree
dnf check-update
```

- **语法**：`dnf install [-y] 包名` / `dnf remove [-y] 包名` / `dnf check-update`
- **作用**：装、卸、只检查可用更新（不安装）

实测：

- `dnf info tree` 显示 **tree 已安装**（来自 `anaconda`，即系统安装时自带）→ `dnf install -y tree` 返回"已安装，无需任何处理"
- 随后 `dnf remove tree` **真的把它删了**（释放 106k）→ 如果还要用，`sudo dnf install -y tree` 可重装
- `dnf check-update`：可用更新非常多（NetworkManager、kernel、openssh、systemd、glibc 等 100+ 包），说明系统自 10/28 安装后**没有全量更新过**；本次**未升级**（保守做法正确）

> 建议：学习机上可以保持现状；真要升级先 `dnf update --security` 或分批，内核升级后需重启。

---

## 四、磁盘（#43 #44 #46 #48）

```bash
df -h
df -i
du -h --max-depth=1 /var | sort -h | tail -5
du -sh /var/log
lsblk
lsblk -f
sudo find / -xdev -type f -size +100M 2>/dev/null
```

- **语法/作用**：
  - `df -h` 分区使用率；`df -i` inode 使用率
  - `du -sh 目录` / `du --max-depth=1` 算目录大小（读不了的目录会报 Permission denied，可加 `sudo` 或 `2>/dev/null`）
  - `lsblk` 块设备树；`-f` 带文件系统与 UUID
  - `find -xdev -size +100M` 找大文件

实测：

```
/dev/mapper/rl-root   36G  4.1G   32G   12%  /
/dev/sda1           1014M  211M  804M   21%  /boot
```

- inode：根分区 57324/18386944 = **1%**，/boot 311/524288 = 1%（没有 inode 压力）
- `/var` 共 **723M**：`/var/cache` 237M、`/var/tmp` 233M、`/var/log` 134M、`/var/lib` 120M
- 磁盘布局：`sda` 40G → `sda1` 1G `/boot` + `sda2` 39G（LVM：`rl-root` 35.1G `/`、`rl-swap` 3.9G swap）；`sr0` 是 3.1G 的安装 ISO
- 超过 100M 的文件只有 2 个：`/var/lib/rpm/Packages`、`/usr/lib/locale/locale-archive`

> 判断思路：`df` 看哪个分区满 → `du` 定位大目录 → `find` 找具体大文件；`df -i` 排除"小文件把 inode 用光"。

---

## 五、内存与挂载（#45 #47）

```bash
free -h
swapon --show
mount | head
findmnt
cat /etc/fstab
```

- **语法/作用**：
  - `free -h`：内存/交换；重点看 **available**（真正可分配），`buff/cache` 可回收
  - `swapon --show`：交换设备
  - `mount`：已挂载列表；`findmnt` 树形展示
  - `/etc/fstab`：开机自动挂载配置（只读，别改）

实测：

```
Mem: 3.5Gi total, 287Mi used, 2.4Gi free, 861Mi buff/cache, 3.0Gi available
Swap: 3.9Gi, 0B used
```

`/etc/fstab` 三行：`rl-root /`、`UUID=1e3e511e-… /boot`、`rl-swap none swap`。

> 注意 `df -h` 里的 `devtmpfs`/`tmpfs` 是**内存文件系统**（/dev、/dev/shm、/run），不占磁盘。

---

## 六、命令语法与作用（汇总）

| 命令 | 语法 | 作用 |
|---|---|---|
| `rpm` | `rpm -q/-qa/-qi/-ql/-qf [-qR]` | 查询已装包、包信息、包文件、文件归属 |
| `dnf repolist` | `dnf repolist` | 列出可用仓库 |
| `dnf provides` | `dnf provides 文件/命令` | 按文件反查包 |
| `dnf info` | `dnf info 包名` | 包详情（版本/仓库/说明） |
| `dnf install/remove` | `dnf install\|remove [-y] 包名` | 安装 / 卸载 |
| `dnf check-update` | `dnf check-update` | 只查可用更新，不安装 |
| `dnf history` | `dnf history [info ID]` | 查看/回滚事务历史 |
| `df` | `df -h` / `df -i` | 分区空间 / inode 使用率 |
| `du` | `du -sh 目录` / `--max-depth=1` | 目录大小 |
| `lsblk` | `lsblk [-f]` | 块设备树 / 文件系统信息 |
| `find -size` | `find 路径 -xdev -type f -size +100M` | 找大文件 |
| `free` | `free -h` | 内存与交换 |
| `mount` / `findmnt` | `mount \| head` / `findmnt` | 挂载列表 / 树形挂载 |

---

## 七、结论与坑

1. 系统健康：根分区 12%、inode 1%、内存 available 3.0G、swap 0 使用、大文件只有 2 个。
2. **531 个包**；tree 原本就是系统自带的（anaconda），`install` 是空操作，`remove` 真删了——要恢复就 `sudo dnf install -y tree`。
3. `dnf check-update` 显示 100+ 更新（含 kernel/openssh/systemd），未升级；升级前建议 `dnf update --security` 并安排重启。
4. 三个手误值得记：`rpm -qf /usr/sbin /sshd`（应连写一个路径）、`lsbik`（应 `lsblk`）、`cat /ect/fstab`（应 `/etc/fstab`）。
5. `du` 读 `/var/lib`、`/var/log/audit` 等要 `sudo`，否则 Permission denied 会让统计偏小。

---

## 八、遗留（沿用）

- [ ] 用户名改名 rocky → cys（需控制台/临时管理员）
- [ ] crash 记录持续观察（`last -x | head`）
