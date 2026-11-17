# 任务2：爆破源 Top5 统计

> 2026-11-17 · Rocky Linux 8.10（rocky@cycs）· 手账"三个真任务"第 2 题
> 数据：真实 sshd 日志（`journalctl -u sshd`）+ 格式一致的练习日志 `/tmp/sshd-test.log`

## 一、目的

从 sshd 的失败登录日志里统计**高频来源 IP**——安全运营/应急响应最基础的一步：定位攻击源、判断是误报还是爆破，为告警、封禁、加白提供依据。

## 二、命令

核心管道（真实日志与练习日志通用）：

```bash
journalctl -u sshd --since today -o cat \
  | grep 'Failed password' \
  | awk '{print $(NF-3)}' \
  | sort | uniq -c | sort -rn | head -5
```

各段职责：筛失败行 → 抠倒数第 3 列（来源 IP）→ 排序 → 相邻去重计数 → 按次数倒序 → 取前 5。

**真实结果**（本机 3 次故意输错密码的尝试）：

```
3 10.0.0.150
```

**练习数据结果**（`/tmp/sshd-test.log`，60 行、7 个来源 IP）：

```
9 203.0.113.5
9 203.0.113.4
9 203.0.113.3
9 203.0.113.2
8 203.0.113.7
```

> 60 = 4×9 + 3×8：4 个 IP 各 9 次、3 个 IP 各 8 次；`sort -rn` 在次数并列时按整行倒序，所以只显示了其中一个 8 次 IP。

进阶维度：

```bash
# 按小时（$3 是时间字段，cut 取小时）
grep 'Failed password' /tmp/sshd-test.log | awk '{print $3}' | cut -d: -f1 | sort | uniq -c

# 按被撞用户名（$11；真实日志中是无效用户名的名字）
grep 'Failed password' /tmp/sshd-test.log | awk '{print $11}' | sort | uniq -c | sort -rn
```

## 三、遇到的问题

1. **`/var/log/secure` 不存在**——Rocky 8.10 最小化安装没装 rsyslog，日志全在 systemd-journald 里，传统 syslog 路径不可用。
2. **journald 默认不持久**——`/var/log/journal` 不存在时日志只存内存，重启即丢；11/15 的 6 次失败记录就是这样没的。
3. **journalctl 输出带前缀**——每行是 `月 日 时间 主机 进程:` 五个字段，`$4` 是主机名、`$6` 是消息首词，直接按"第几个字段"取 IP 会取错。
4. **真实来源太少**——本机练习只有 1 个来源 IP，看不出 Top5 的排序效果。

## 四、怎么解决

1. 启用 journald 持久化并验证：

   ```bash
   sudo mkdir -p /var/log/journal
   sudo systemd-tmpfiles --create --prefix /var/log/journal
   sudo systemctl restart systemd-journald
   journalctl --disk-usage     # 16.0M，日志已落盘
   ```

2. 统一用 `journalctl -o cat` 去掉前缀，或改用**从末尾数**的 `$(NF-3)`——两种写法都不怕前缀变化。
3. 真实日志先跑通链路，再生成**格式一致**的练习日志补充多来源场景（练习数据必须显式标注，不能当作真实发现）：
   ```bash
   : > /tmp/sshd-test.log
   for i in $(seq 1 60); do
     ip="203.0.113.$((i % 7 + 1))"
     printf 'Nov 17 09:%02d:01 cycs sshd[%d]: Failed password for invalid user admin from %s port %d ssh2\n' \
       $((i % 60)) $((1000 + i)) "$ip" $((40000 + i)) >> /tmp/sshd-test.log
   done
   ```

## 五、结论与下一步

- 一条管道就能完成"**谁在撞、撞了多少次**"的基本研判：`3 10.0.0.150` 是自己 ssh 错用户名产生的，属误报；换成真实攻击时会是一串外网 IP。
- 记忆点：`uniq` 前必须 `sort`（只合并相邻行）；`-o cat` / `$(NF-k)` 是日志切字段的保命写法；没有持久化日志就没有取证。
- 下一步：Day 14 会用到 `tail -f` / `last` / `lsof`，把"实时监控 + 登录历史 + 端口占用"接进同一套日志分析流程。
