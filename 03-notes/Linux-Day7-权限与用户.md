# Linux Day 7：权限与用户（2026-11-18）

> 来源：手机端整理（2026-11-18）+ 本机 Rocky 8.10 实测输出。
> 实验环境：`rocky@cycs`，练习目录 `~/text0/perm-lab`；临时用户 `demouser`（实验后已删除）。

---

## 一、建目录与切换

```bash
mkdir -p ~/text0/perm-lab && cd ~/text0/perm-lab
```

- `mkdir -p`：递归创建目录，上级不存在也自动建；目录已存在不报错
- `&&`：前一条命令执行成功，才执行后一条
- `~`：当前用户家目录
- **`mkdir`**：语法 `mkdir [-p] 目录`；作用：创建目录，`-p` 递归建父目录、目录已存在不报错
- **`cd`**：语法 `cd [目录]`；作用：切换当前工作目录（`cd ~` 回家目录，`cd -` 回上一次目录）

---

## 二、umask 权限掩码

- 作用：控制**新建**文件/目录的默认权限（只对新建对象生效，已有文件不受影响）
- 基准：目录上限 `777`、普通文件上限 `666`
- 实测：`umask` = `0002`
  - 新建目录：`777 - 002 = 775` → `drwxrwxr-x`
  - 新建文件：`666 - 002 = 664` → `-rw-rw-r--`
- 临时修改：`umask 022`（只对当前 shell 生效，重开/重启恢复；要永久改得写 `/etc/profile` 或 `~/.bashrc`）
- **`umask`**：语法 `umask [掩码]`；作用：设置/查看新建文件与目录的默认权限掩码（只影响新建对象，仅当前 shell 生效）

> 本机实测输出与公式一致：`f.txt` 为 `-rw-rw-r--`，`sub` 为 `drwxrwxr-x`。

---

## 三、`ls -l` 权限解读（10 位）

```
-rw-r--r--. 1 rocky rocky   0 11月 18 22:06 f.txt
```

| 位置 | 含义 |
|---|---|
| 第 1 位 | 文件类型：`-` 普通文件、`d` 目录、`l` 软链接、`b/c` 设备、`p` 管道、`s` 套接字 |
| 第 2–4 位 | 属主权限 `rwx` |
| 第 5–7 位 | 属组权限 `rwx` |
| 第 8–10 位 | 其他人权限 `rwx` |
| 末尾 `.` | 该文件带 SELinux 安全上下文标签（Rocky 默认开启） |
- **`ls`**：语法 `ls [-l] [路径]`；作用：列目录内容，`-l` 长格式显示权限/属主/大小/时间

---

## 四、`chmod` 修改权限（rwx）

```bash
chmod 600 f.txt          # 数字模式：属主/组/其他人
chmod 640 f.txt
chmod u+x,g-w f.txt      # 符号模式
chmod 755 sub
chmod -R 700 sub        # -R 递归目录内全部内容
```

- 数字模式：三位八进制，分别对应属主、属组、其他人
- 符号模式：`u` 属主、`g` 属组、`o` 其他人、`a` 全部；`+` 加、`-` 减、`=` 设定
- `-R`（大写 R）：递归，目录 + 内部所有文件/子目录一起改
- **`chmod`**：语法 `chmod [-R] 权限 目标`（八进制 `755` 或符号 `u+x,g-w`）；作用：修改权限位（rwx）

> 实测：`chmod 600 f.txt` 后变 `-rw-------`；`chmod 755 sub` 后变 `drwxr-xr-x`。

---

## 五、`chown` 修改属主 / 属组

```bash
sudo chown root:root f.txt && ls -l f.txt
sudo chown rocky:rocky f.txt
sudo chown -R rocky:rocky ~/text0/perm-lab
```

- 语法：`chown 属主:属组 目标`
- 简写：`chown rocky f.txt` 只改属主；`chown :rocky f.txt` 只改属组
- **需要 root**：普通用户不能把文件"送"给别人，改属主必须 `sudo`
- `-R` 递归修改目录内全部内容
- 核对：`ls -l` 第 3 列属主、第 4 列属组
- **`chown`**：语法 `chown [-R] 属主[:属组] 目标`；作用：修改文件属主/属组（需要 root 权限）

> 实测：`sudo chown root:root f.txt` 后第 3、4 列变成 `root root`；改回 `rocky:rocky` 正常。

---

## 六、用户管理：`useradd / usermod / passwd / chpasswd`

```bash
sudo useradd -m demouser                  # -m 建家目录
echo 'demouser:Test@1234' | sudo chpasswd # 非交互式设密码
id demouser                               # uid / gid / 所属组
sudo usermod -aG wheel demouser           # -aG 追加附加组（wheel = sudo 管理员组）
sudo -l -U demouser                       # 查看该用户的 sudo 授权范围
sudo passwd -S demouser                   # 查密码状态
sudo userdel -r demouser                  # 清理：删用户 + 家目录
```

**命令语法与作用**

- **`useradd`**：语法 `useradd [-m] [-u UID] 用户名`；作用：新建用户（`-m` 同时建家目录）
- **`chpasswd`**：语法 `echo '用户:密码' | chpasswd`；作用：非交互批量设密码（密码不出现在命令行）
- **`usermod`**：语法 `usermod -aG 组 用户`；作用：修改已有用户属性（追加附加组、改 shell、锁账号等）
- **`passwd`**：语法 `passwd [-S | -l | -u] [用户]`；作用：改密码；`-S` 查状态、`-l` 锁定、`-u` 解锁
- **`id`**：语法 `id [用户]`；作用：显示 uid/gid 与附加组
- **`sudo`**：语法 `sudo [-u 用户] 命令`；作用：以其他用户（默认 root）身份执行单条命令
- **`userdel`**：语法 `userdel [-r] 用户`；作用：删除用户，`-r` 连同家目录

### 重点参数

| 参数 | 含义 |
|---|---|
| `useradd -m` | 自动创建 `/home/用户名`；不加 `-m` 不会建 |
| `usermod -aG 组 用户` | `-a` 是**追加**；去掉 `-a` 会**覆盖**原有附加组（会把 wheel 弄丢） |
| `chpasswd` | 非交互式设密码（`用户:密码` 从 stdin 读），适合脚本 |
| `passwd -S` | 查询密码状态（P 可用 / L 锁定 / NP 无密码） |
| `passwd -l` | 锁定密码（禁止登录） |
| `passwd -u` | 解锁密码 |

### 实测结果

- `id demouser` → `uid=1001(demouser) gid=1001(demouser) 组=1001(demouser),10(wheel)`（加入 wheel 生效）
- `sudo -l -U demouser` → `用户 demouser 可以在 cys 上运行以下命令： (ALL) ALL`
- `passwd -S demouser` → `demouser PS 2026-11-18 0 99999 7 -1 (密码已设置，使用 SHA512 算法。)`
  - 括号里明确写了"密码已设置，使用 SHA512 算法"；状态列常见取值是 P（可用）/ L（锁定）/ NP（无密码），本机这行显示为 `PS`，具体含义以 `man passwd` 为准
- `sudo userdel -r demouser` 后 `id demouser` → `无此用户`，清理完成

---

## 七、核心区分总表

| 命令 | 改什么 |
|---|---|
| `chmod` | 改权限位（rwx） |
| `chown` | 改文件属主 / 属组 |
| `useradd` | 新建用户 |
| `usermod` | 修改已有用户信息（加组、改 shell、锁账号…） |
| `umask` | 控制新建文件/目录的默认权限 |

---

## 八、今天实测踩到的点

1. `sudo grep -n wheel /etc/sudoers` → 第 107 行 `%wheel ALL=(ALL) ALL` 是**启用**的；第 108 行 `#%wheel ALL=(ALL) NOPASSWD: ALL` 是示例注释（前面 `#` 等于没启用）。
2. `chown` 不带 `sudo` 会失败：普通用户无权改属主。
3. `usermod -aG` 漏掉 `-a` 会覆盖附加组——这是最危险的一个坑（对 rocky 这么做会丢 wheel，需要重新登录后补加）。
4. 组变更要**重新登录**才对已存在的会话生效。
5. `passwd -S` 比 `passwd` 适合"只看状态"，后者会进入交互式改密流程。
