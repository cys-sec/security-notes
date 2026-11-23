# Linux Day 12：网络（练）——端口基线 + 接口探测（2026-11-23）

> 环境：Rocky 8.10（rocky@rocky），NAT 网段；证据：终端截图（2026-11-23 22:36–22:53）。

---

## 一、监听端口基线（ss）

```bash
sudo ss -tlnp
sudo ss -tulnp
ss -tlnp | grep -v ':22'
```

- **语法**：`ss [选项]`；`-t` TCP、`-u` UDP、`-l` 只监听、`-n` 数字端口、`-p` 显示进程（需 root）
- **作用**：列出监听端口与进程；用 `grep -v ':22'` 检查"除 22 之外还开了什么"

实测：

```
LISTEN 0 128 0.0.0.0:22 0.0.0.0:* users:(("sshd",pid=1011,fd=3))
LISTEN 0 128   [::]:22    [::]:* users:(("sshd",pid=1011,fd=4))
```

`grep -v ':22'` **无输出** → 基线干净：只有 22 在监听。

`ss -tulnp` 里还看到 UDP 的 DHCP 会话：

```
udp ESTAB 10.0.0.150%ens160:68  10.0.0.254:67  users:(("NetworkManager",pid=995,fd=26))
tcp ESTAB 10.0.0.150:22         10.0.0.1:59860  users:(("sshd",pid=1802,fd=4))
```

> DHCP 客户端（68）向 67 端口要地址 → 地址是 `10.0.0.254` 发的；TCP 那条就是你的 FinalShell 连接。

---

## 二、curl 的三种结果（今天最有价值）

### 1）正常 HTTP：200

```bash
curl -sS -o /dev/null -w '%{http_code} %{time_total}\n' --connect-timeout 3 https://example.com
```

- **语法**：`curl [-sS] [-o 文件] [-w 格式] [--connect-timeout 秒] URL`
- **作用**：只要"状态码 + 耗时"，适合巡检

实测：`200 2.130529`；带更多字段时：`code=200 ip=104.20.23.154 time=2.503422s size=577`。

### 2）端口在听、但不是 HTTP：协议不匹配

```bash
curl -v --connect-timeout 3 http://127.0.0.1:22
```

实测输出：

```
* Connected to 127.0.0.1 (127.0.0.1) port 22 (#0)
> GET / HTTP/1.1
...
* SSH-2.0-OpenSSH_8.0
* Invalid SSH identification string.
* Recv failure: Connection reset by peer
curl: (56) Recv failure: Connection reset by peer
```

> **TCP 连上了**（`Connected`），但对面是 SSH，收到的是 SSH banner，不是 HTTP 响应 → 连接被重置。这证明 **"端口通 ≠ 服务可用"**。

### 3）端口没监听：拒绝

```bash
curl --connect-timeout 3 http://127.0.0.1:9
```

实测：`curl: (7) Failed to connect to 127.0.0.1 port 9: Connection refused` —— 立刻被拒绝（RST），和"超时（被丢弃）"完全不同。

| 现象 | 含义 |
|---|---|
| `200` | 服务正常，应用层可用 |
| `Connected` 后协议错/重置（56） | 端口在听，但服务不是你要的协议 |
| `Connection refused`（7） | 主机可达，端口没服务 |
| `Connection timed out`（7） | 包被丢弃（防火墙/NAT/路由） |

---

## 三、串成巡检链

```bash
sudo ss -tlnp
systemctl status sshd --no-pager
journalctl -u sshd -n 10 --no-pager
```

- **语法/作用**：`ss` 看端口 → `systemctl status` 看服务 → `journalctl -u` 看日志；三者一致才叫"确认服务正常"

实测：

- `systemctl status sshd`：`active (running) since Mon 2026-11-23 17:48:16 CST; 4h 41min ago`，`Main PID: 1011`
- `journalctl -u sshd -n 10`：11/22 两次 `Accepted`（来自 10.0.0.1）→ `-- Reboot --` → 11/23 17:48 sshd 启动 + 一条 `Accepted`

---

## 四、命令语法与作用（汇总）

| 命令 | 语法 | 作用 |
|---|---|---|
| `ss` | `ss [-tulnps]` | 监听端口 / 已建立连接 / 汇总 |
| `curl` | `curl [-sS] [-o 文件] [-w 格式] [--connect-timeout 秒] URL` | HTTP 探测：状态码、耗时、远端 IP |
| `systemctl` | `systemctl status 单元 --no-pager` | 服务状态 + 最近日志 |
| `journalctl` | `journalctl -u 单元 -n 行数 --no-pager` | 按服务查日志 |

---

## 五、结论与坑

1. **基线只有 22**（sshd），DHCP 走 UDP 68→67，其余没有监听端口——干净。
2. `0.0.0.0:22` 与 `[::]:22` 表示所有网卡可达（IPv4/IPv6），`127.0.0.1:22` 才是仅本机。
3. **三种 curl 结果要能区分**：200 / 协议不匹配(56) / 拒绝(7) / 超时(7)。
4. 今天 17:48 又有一次 `-- Reboot --`：如果并非你主动重启，继续留意 crash 问题（`last -x | head` 看是否有 `crash`）。

---

## 六、遗留（沿用）

- [ ] 用户名改名 rocky → cys（需控制台/临时管理员）
- [ ] crash 记录持续观察（`last -x | head`、`journalctl -b -1 -p err`）

---

## 七、手机端知识点汇总（2026-11-23）

> 来源：手机端整理（当日晚）；已与本笔记合并。⚠️ 为补充/修正。

### 1. ss / 网络连接查看

- `ss -tlnp`：`Local Address:Port` 是本机端、`Peer Address:Port` 是对端
- `0.0.0.0:22` = 所有 IPv4 接口监听 22；`[::]:22` = 所有 IPv6 接口
- `LISTEN` 监听、`ESTAB` 连接已建立
- `ss -tunp` 输出里的 `Netid` 列 = 协议标识（`tcp` / `udp`），区分连接类型
- SSH 那条 ESTAB：Local `10.0.0.150:22`、Peer `10.0.0.1:59860`、进程 `sshd`

### 2. IP、邻居与路由

- `ip neigh`（可简写 `ip n`）：同网段邻居表（IP ↔ MAC）
  - `REACHABLE` = 二层已拿到 MAC、链路可达；`INCOMPLETE` = ARP 无回应
- ⚠️ 二层可达 **≠** IP/DNS/应用一定通；反过来"ping 不通"也不代表网络断——**本机实测：`ping 10.0.0.1` 100% 丢包，但 SSH 连接正常**（ICMP 被宿主机防火墙拦）
- `ip route get 8.8.8.8`：查实际路由路径，出现 `via` 就是网关
- ⚠️ **修正**：这台机器的网关是 **`10.0.0.2`**（VMware NAT：`default via 10.0.0.2`），`10.0.0.1` 是宿主机（VMnet8 网卡），**不是网关**；判断网关一律以 `ip route` 的 `default via` 为准，别按"同网段 .1 就是网关"猜
- 网关是内网出口；跳出公网 IP 才算出外网 ✓

### 3. traceroute

- 输出：跳号 + IP + 三次时延；`* * *` = 该跳无应答
- 第一跳内网 IP 通常是网关；后面出现公网 IP 代表离开内网
- ⚠️ 本机实测 `traceroute -n 10.0.0.1` 全 `* * *`：ICMP/UDP 被过滤时就是这样，换 `-T`（TCP 探测）或 `tracepath`
- traceroute 通 ≠ DNS 通：它走 IP/ICMP，DNS 是另一个服务 ✓

### 4. curl 客户端测试

- `-I` 只拿响应头；`-sS` 静默但保留错误；`-o /dev/null` 丢弃正文；`-w '...\n'` 自定义输出
- `%{http_code}` 状态码、`%{time_total}` 总耗时（秒）、`\n` 换行、`--connect-timeout n` 连接超时
- `-v` 详细调试：依次看到 DNS → TCP → TLS → HTTP
- 输出 `200 2.13` = HTTP 请求成功 + 总耗时

### 5. 排查链路（分层）

- 精简链：`ping → ip route get → traceroute / ip neigh → getent → curl -I`
- 分层理解：
  - `ping`：IP 是否可达（注意 ICMP 可能被过滤）
  - `ip route get`：路由对不对、走哪个网关
  - `traceroute`：卡在哪一跳；`ip neigh`：二层邻居/MAC
  - `getent`：域名 DNS 解析
  - `curl`：HTTP/HTTPS 应用层验证
- 一句话串：网卡/二层邻居 → 路由网关 → DNS → 逐跳 → TCP 端口 → curl 应用层
