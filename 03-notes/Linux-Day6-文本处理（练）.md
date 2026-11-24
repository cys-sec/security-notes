# Linux Day 6：文本处理（练）——爆破源 Top5（2026-11-17）

> 环境：Rocky Linux 8.10，用户 `rocky`；数据：真实 sshd 日志（`journalctl -u sshd`）+ 格式一致的练习日志 `/tmp/sshd-test.log`。
> 来源：`02-security/任务2-爆破源Top5.md`（当时完整记录）。

---

## 一、今天的目标

把 Day5 的「筛 → 抠 → 排 → 数」管道，用在**真实安全场景**上：从失败登录日志里找出高频来源 IP，判断是误报还是爆破。

## 二、核心管道

```bash
journalctl -u sshd --since today -o cat \
  | grep 'Failed password' \
  | awk '{print $(NF-3)}' \
  | sort | uniq -c | sort -rn | head -5
```

各段职责：

| 段 | 作用 |
|---|---|
| `journalctl -u sshd --since today -o cat` | 取今天的 sshd 日志，`-o cat` 去掉前缀 |
| `grep 'Failed password'` | 只留失败登录行 |
| `awk '{print $(NF-3)}'` | 从末尾数第 3 列 = 来源 IP（不受前缀长度影响） |
| `sort \| uniq -c` | 排序后去重计数 |
| `sort -rn \| head -5` | 按次数倒序取前 5 |

## 三、实测结果

### 1. 真实日志（本机）

```text
      3 10.0.0.150
```

**只有 1 个来源、3 次**——是自己在 `ssh` 时故意输错用户名产生的，**属误报**。
> 判断依据：来源是本机地址、次数极少、无陌生外网 IP。

### 2. 练习日志（补充多来源场景）

用格式一致的 60 行日志（7 个来源 IP、`203.0.113.x` 为保留测试网段）：

```text
      9 203.0.113.5
      9 203.0.113.4
      9 203.0.113.3
      9 203.0.113.2
      8 203.0.113.7
```

> 60 = 4×9 + 3×8：4 个 IP 各 9 次、3 个 IP 各 8 次；`sort -rn` 在次数并列时按整行倒序，所以只显示了其中一个 8 次 IP。
> **练习数据必须显式标注，不能当作真实发现。**

### 3. 进阶维度

```bash
# 按小时分布（$3 是时间字段，cut 取小时）
grep 'Failed password' /tmp/sshd-test.log | awk '{print $3}' | cut -d: -f1 | sort | uniq -c

# 被撞的用户名（$11）
grep 'Failed password' /tmp/sshd-test.log | awk '{print $11}' | sort | uniq -c | sort -rn
```

## 四、实践反馈（4 个真问题）

| # | 问题 | 原因 | 解决 |
|---|---|---|---|
| 1 | `/var/log/secure` 不存在 | Rocky 8.10 最小化没装 rsyslog，日志全在 journald | 统一用 `journalctl` |
| 2 | 查 11/15 的日志返回 0 行 | **journald 默认不持久**，日志只在内存，重启即丢 | 开持久化（见下） |
| 3 | 直接按"第几个字段"取 IP 取错 | `journalctl` 每行有 `月 日 时间 主机 进程:` 前缀 | `-o cat` 去前缀，或从末尾数 `$(NF-3)` |
| 4 | 真实日志来源太少，看不出 Top5 效果 | 单人练习环境 | 生成**格式一致**的练习日志补场景（并标注） |

开持久化（可复现）：

```bash
sudo mkdir -p /var/log/journal
sudo systemd-tmpfiles --create --prefix /var/log/journal
sudo systemctl restart systemd-journald
journalctl --disk-usage          # 实测 16.0M，日志已落盘
```

## 五、结论

- 一条管道就能完成"**谁在撞、撞了多少次**"的最小研判：`3 10.0.0.150` 是自己撞自己（误报）；真实攻击会是一串外网 IP
- 记忆点：**`uniq` 前必须 `sort`**；`-o cat` / `$(NF-k)` 是日志切字段的保命写法
- **没有持久化日志就没有取证**——这是安全运营的基本前提

## 六、和前后几天的联系

- Day5 练的是命令，今天练的是**把命令组装成研判能力**
- Day9/Day10 会继续这条线：`journalctl --failed`、`last/lastb` 登录审计
