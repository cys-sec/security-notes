# Linux Day 14：压缩 + 日志安全（2026-11-25）

> 环境：Rocky 8.10（rocky@rocky）；证据：终端截图（2026-11-25 21:15–22:52）。
> 本日完成 **任务 3（打包 + scp 传输 + md5 校验）**；仅 `rsync` 未跑（待补）。

---

## 一、tar / gzip / zip（#49 #50 #51）

```bash
cd ~
tar -czf /tmp/text0.tar.gz text0
echo $?
tar -tzf /tmp/text0.tar.gz | head
```

- **语法/作用**：`tar -czf 目标 源`（创建 gzip 压缩包）；`-t` 列表；`-x` 解压
- 坑 1：在 `~/text0` 里用 `text0*` → shell 无匹配，原样传给 tar → `无法 stat`；从 `~` 用 `text0` 或 `cd ~/text0` 用 `.`
- 坑 2：`text0/etc-full/shadow`、`gshadow` 权限是 `----------`（0000），属主 rocky 也读不了 → 改用 `sudo tar -czf …`
- 坑 3：**tar 报错仍会生成"半成品包"**，必须 `echo $?` 确认 0，并用 `tar -tzf | wc -l` 核对

```bash
mkdir -p /tmp/restore && sudo tar -xzf /tmp/text0.tar.gz -C /tmp/restore/
```

- **语法/作用**：`-x` 解压、`-C 目录` 指定目标；sudo 打包时也用 sudo 解压，权限/属主才能还原

```bash
gzip -k /tmp/text0.tar.gz 2>/dev/null || true
sudo zip -r /tmp/text0.zip text0*
unzip -l /tmp/text0.zip | head
```

- **语法/作用**：`gzip 文件`（单文件压缩，`-k` 保留原文件）；`zip -r`/`unzip -l`（zip 与 Windows 通用）
- zip 同样栽在 shadow 权限上：普通用户 4 条 `Permission denied`，`sudo zip` 才完整

---

## 二、scp + md5 校验（#52，任务 3 核心）

```bash
md5sum /tmp/text0.tar.gz
scp /tmp/text0.tar.gz rocky@10.0.0.150:/tmp/
ssh rocky@10.0.0.150 'md5sum /tmp/text0.tar.gz'
```

- **语法/作用**：`scp 文件 用户@主机:路径`（基于 SSH 的**全量**复制）；`ssh 主机 '命令'` 远程执行
- 实测：本地 `4fad6d39394eb50a741e49ffc9f93f75`，对端 `4fad6d39394eb50a741e49fc9f93f75` → **一致，传输完整**；scp 显示 `12MB 118.5MB/s`
- 第一次 `scp` 时看到 usage 提示（多半是命令没敲全），重敲后正常

---

## 三、ssh-keygen + 免密登录（#54）

```bash
ls ~/.ssh
ssh-keygen -t ed25519 -f ~/.ssh/id_ed25519_lab -N '' -C 'lab-key'
cat ~/.ssh/id_ed25519_lab.pub
ssh-copy-id -i ~/.ssh/id_ed25519_lab.pub rocky@10.0.0.150
ssh -i ~/.ssh/id_ed25519_lab rocky@10.0.0.150 'whoami'
```

- **语法/作用**：`ssh-keygen -t 类型 -f 文件 -N 密码 -C 注释`；`ssh-copy-id` 把公钥写入对端 `~/.ssh/authorized_keys`；`ssh -i 私钥` 指定私钥登录
- 原有 `~/.ssh` 只有 `authorized_keys`、`known_hosts`，没有默认私钥 → 用新文件名 `id_ed25519_lab`，**没有覆盖任何东西**
- 公钥指纹：`SHA256:31APnUj2/KyqXUvaHLBdoXLSKliTjB7jK6tFpMJH6I`
- 免密登录成功：`ssh -i … 'whoami'` → `rocky`
- journal 里对应记录：`Accepted publickey for rocky from 10.0.0.150 port 48230 ssh2: ED25519 SHA256:31APnUj2…` ——**指纹和生成时一致**，证明用的就是这把钥匙

---

## 四、日志实时监控与筛选（#55 #56）

```bash
journalctl -f -u sshd
```

- **语法/作用**：`-f` 实时跟随（这台没有 `/var/log/secure`，只能用 journald）；`Ctrl+C` 退出

```bash
journalctl -u sshd --since today | grep -c 'Failed password'
journalctl -u sshd --since today | grep 'Accepted' | tail -3
```

- **语法/作用**：`grep -c` 计数、`grep … | tail -3` 看最近几条
- 实测：今天 `Failed password` = **0**；`Accepted` 能看到 `password` 和 `publickey` 两种认证方式

---

## 五、登录审计与端口占用（#57 #58 #59 #60）

```bash
last -n 10
w
sudo lsof -i :22
sudo ss -tulnp
```

- **语法/作用**：`last`（wtmp 成功登录历史）；`w`（当前会话 + 负载 + 正在执行命令）；`lsof -i :22`（谁占用 22 端口：监听 + 已建立连接）；`ss -tulnp`（端口与进程）
- `lsof -i :22` 输出里同时有 `LISTEN`（sshd 1015）和多条 `ESTABLISHED`（你几个 FinalShell 会话）
- `ss -tulnp` 补充发现：**chronyd 监听 UDP 323**（`127.0.0.1:323`、`[::1]:323`）——之前只看 TCP 22 时会漏掉它

---

## 六、命令语法与作用（汇总）

| 命令 | 语法 | 作用 |
|---|---|---|
| `tar` | `tar -czf 包 源` / `-tzf` / `-xzf -C` | 打包（可 gzip）、列表、解压 |
| `gzip` | `gzip [-k] 文件` | 单文件压缩 |
| `zip`/`unzip` | `zip -r 包 源` / `unzip -l 包` | 跨平台压缩/列表 |
| `scp` | `scp 文件 用户@主机:路径` | 基于 SSH 的全量复制 |
| `rsync` | `rsync -av 源/ 目标/` | 增量同步（**待做**） |
| `ssh-keygen` | `ssh-keygen -t ed25519 -f 文件 -N ''` | 生成密钥对 |
| `ssh-copy-id` | `ssh-copy-id -i 公钥 用户@主机` | 安装公钥实现免密 |
| `journalctl` | `-f` / `-u 单元` / `--since` | 实时/按服务/按时间查日志 |
| `grep` | `grep -c / -E / -oE` | 统计/正则/只取匹配片段 |
| `last` | `last -n N` | 成功登录历史 |
| `w` | `w` | 当前会话与负载 |
| `lsof` | `lsof -i :端口` | 谁占用了端口/文件 |
| `ss` | `ss -tulnp` | 端口与进程 |

---

## 七、结论与坑

1. **任务 3 完成**：tar 打包 → scp 传输 → 两端 md5 一致（`4fad6d39…`）。
2. **影子文件权限 0000**：普通用户（含属主）读不了，tar/zip 必须 `sudo`；这也是"shah 文件不能随便看"的权限体现。
3. **压缩失败仍出包**：一定要检查 `$?` 与包内条目数，别把半成品当备份。
4. **密钥指纹可核对**：`ssh-keygen` 输出的 `SHA256:31APnUj2…` 与 journal 里 `Accepted publickey … ED25519 SHA256:31APnUj2…` 一致。
5. **端口基线要包含 UDP**：TCP 22 之外还有 chronyd 的 UDP 323（仅本机）。
6. `last` 里仍能看到 **crash** 记录（今天 12:44 前后），系统还有非正常关机，建议继续观察。

---

## 八、遗留

- [ ] `rsync -av ~/text0/ /tmp/rsync-test/`（#53）
- [ ] 用户名改名 rocky → cys
- [ ] crash 记录观察（`last -x`、`journalctl -b -1 -p err`）
