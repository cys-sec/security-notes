# Linux Day 11：网络（2026-11-22）

> 环境：Rocky 8.10（rocky@rocky），VMware NAT 网段；证据：终端截图（2026-11-22 22:23–22:55）。
> 网络：`ens160` = `10.0.0.150/24`，网关/DNS = `10.0.0.2`，宿主机 = `10.0.0.1`。

---

## 一、ip —— 地址、路由、网卡

```bash
ip -4 addr show
ip route
ip link
ip neigh
```

- **语法**：`ip [对象] [动作]`（对象：`link`/`addr`/`route`/`neigh`）
- **作用**：看网卡状态、IP 地址、路由表、ARP 邻居表

实测：

```
2: ens160: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 state UP
    link/ether 00:0c:29:fd:22:23
    inet 10.0.0.150/24 brd 10.0.0.255 scope global dynamic noprefixroute ens160
default via 10.0.0.2 dev ens160 proto dhcp src 10.0.0.150 metric 100
nameserver 10.0.0.2                     # /etc/resolv.conf
```

邻居表：

```
10.0.0.1    dev ens160 lladdr 00:50:56:c0:00:08 REACHABLE   # 宿主机（VMnet8 主机网卡）
10.0.0.2    dev ens160 lladdr 00:50:56:e1:2a:1e REACHABLE   # VMware NAT 网关
10.0.0.254  dev ens160 lladdr 00:50:56:f1:dd:fa STALE
```

> 结论：VM 通过 `ens160` 走 NAT 上网，网关和 DNS 都是 `10.0.0.2`；`10.0.0.1` 是宿主机，`10.0.0.254` 可能是 DHCP/其他虚拟网卡。

---

## 二、ss —— 监听端口与连接

```bash
ss -tnp
ss -s
```

- **语法**：`ss [选项]`；`-t` TCP、`-l` 只监听、`-n` 数字端口、`-p` 显示进程、`-s` 汇总
- **作用**：看已建立连接（不带 `-l`）和监听端口（带 `-l`）；`-p` 显示进程需要权限

实测：

```
State  Recv-Q Send-Q Local Address:Port   Peer Address:Port
ESTAB  0      0      10.0.0.150:22         10.0.0.1:64255
```

> 这就是你 FinalShell 的 SSH 连接（宿主机 `10.0.0.1` → VM `22` 端口）。`ss -p` 没显示进程名，因为 sshd 是 root 的进程、普通用户看不到——**要 `sudo ss -tnp`**。

```text
Total: 124
TCP: 3 (estab 1, closed 0, orphaned 0, timewait 0)
```

> `ss -s` 给汇总：TCP 3 个、其中 1 个已建立；数字小说明系统很干净。

---

## 三、ping —— 连通性（今天最大的"坑"）

```bash
ping -c 3 127.0.0.1        # 回环：0% 丢包，0.069ms
ping -c 3 10.0.0.1         # 宿主机：100% 丢包！
ping -c 3 -W 1 8.8.8.8     # 外网：0% 丢包，312–321ms
```

- **语法**：`ping [-c 次数] [-W 超时秒] 目标`
- **作用**：测连通性、丢包率、RTT；`-c` 限制次数、`-W` 设超时

**关键结论：`ping 10.0.0.1` 不通 ≠ 宿主机不在。**

- 证据 1：`ip neigh` 里 `10.0.0.1` 是 `REACHABLE`（ARP 通）
- 证据 2：`ss -tnp` 里 SSH 连接来自 `10.0.0.1`，且正常用着
- 原因：**Windows 防火墙默认拦截 ICMP（ping）**，而 TCP 22 放行了

所以"ping 不通"只能说明**ICMP 被拦**，不能说明网络断了——要换 TCP/UDP 测试（`curl`、`ss`、`nc`）。

---

## 四、traceroute —— 路径探测

```bash
which traceroute || echo "没装"
sudo dnf install -y traceroute
traceroute -n 10.0.0.1
```

- **语法**：`traceroute [-n] [-m 跳数] [-T] 目标`；`-n` 不解析域名、`-T` 用 TCP 探测
- **作用**：逐跳显示路径，定位"哪一跳卡住"

实测：`traceroute -n 10.0.0.1` 全是 `* * *`（30 跳全无响应）——**和 ping 一样，ICMP/UDP 被宿主机防火墙拦掉**了。换 `traceroute -n -T 10.0.0.1`（TCP 探测）或 `tracepath` 试试；外网目标可用 `traceroute -n 8.8.8.8`。

---

## 五、dig / getent —— DNS

```bash
dig +short example.com
getent hosts example.com
```

- **语法**：`dig [+short] 域名 [类型]`；`getent hosts 域名`
- **作用**：DNS 解析测试；`dig` 属于 bind-utils（可能没装），`getent` 走系统 NSS、一定可用

实测：`dig` 没装（`-bash: dig: 未找到命令`）；`getent hosts example.com` 正常返回：

```
2606:4700:10::6814:179a  example.com
2606:4700:10::ac42:93f3  example.com
```

> DNS 正常（解析到了 Cloudflare 的 IPv6 地址）。要装 `dig`：`sudo dnf install -y bind-utils`。

---

## 六、curl —— HTTP 客户端

```bash
curl --connect-timeout 3 -I https://example.com
curl -sS -o /dev/null -w '%{http_code}\n' http://10.0.0.1
```

- **语法**：`curl [选项] URL`；`-I` 只看响应头、`-sS` 静默+报错、`-o /dev/null` 丢弃正文、`-w` 输出指定字段、`--connect-timeout` 连接超时
- **作用**：测 HTTP 服务是否可用、状态码、响应头

实测：`curl -I https://example.com` → **`HTTP/2 200`**（外网通，NAT 正常），响应头里有 `server: cloudflare`、`cf-cache-status: HIT`。

`curl http://10.0.0.1` 卡住（你按了两次 `^C`）：宿主机**没有 80 端口的 HTTP 服务**，所以连不上——不是命令写错，而是"目标没有那个服务"；加 `--connect-timeout 3` 会更快返回超时错误（curl 退出码 28）。

---

## 七、命令语法与作用（汇总）

| 命令 | 语法 | 作用 |
|---|---|---|
| `ip` | `ip [link/addr/route/neigh] [show]` | 看网卡、IP、路由、ARP |
| `ss` | `ss [-tlnps]` | 看监听端口与连接 |
| `ping` | `ping [-c N] [-W 秒] 目标` | 测 ICMP 连通性/丢包/延迟 |
| `traceroute` | `traceroute [-n] [-T] 目标` | 逐跳路径探测 |
| `dig` / `getent` | `dig +short 域名` / `getent hosts 域名` | DNS 解析测试 |
| `curl` | `curl [-I] [-sS] [-o 文件] [-w 格式] URL` | HTTP 请求/接口测试 |

---

## 八、结论与坑

1. 拓扑清楚：VM `10.0.0.150/24`，NAT 网关 + DNS 都是 `10.0.0.2`，宿主机 `10.0.0.1`；外网通（ping 8.8.8.8、curl 200）。
2. **ping 不通 ≠ 网络故障**：宿主机防火墙拦 ICMP，但 TCP 22 是通的（`ss` 证据）。判断连通性要交叉验证。
3. **traceroute 全 `*` 同理**：ICMP/UDP 被过滤；换 `-T`（TCP）或 `tracepath`。
4. **`ss -p` 要 root 才看得到进程名**；`curl` 对"没有服务的端口"会卡到超时，加 `--connect-timeout`。
5. `dig` 没装不影响：`getent hosts` 同样能验证 DNS。

---

## 九、遗留（沿用前几日）

- [ ] 用户名改名 rocky → cys（需控制台/临时管理员）
- [ ] 排查 11/20–11/21 的 crash 记录（`journalctl -b -1 -p err`）
