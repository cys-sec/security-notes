# Linux Day 5：文本处理（2026-11-16）

> 环境：Rocky Linux 8.10，用户 `rocky`；素材 `~/text0/etc-full/services`（11473 行）、`/etc/passwd`、`journalctl -u sshd`。
> 来源：`01-linux/命令手账.md` 的「day5 实操」段 + 提交 `5f9adef`（当时真实终端输出）。

---

## 一、今天的目标

安全运维 80% 的日志分析靠这四件套：**筛行（grep）→ 切列（awk/cut）→ 改文本（sed）→ 排序统计（sort/uniq）**。今天用真实文件跑通一遍。

## 二、命令与实测

### 1. `grep` —— 按行筛选

```bash
$ wc -l etc-full/services
11473
$ grep -c '^#' etc-full/services            # 注释行数
107
$ grep -v '^#' etc-full/services | wc -l    # 有效行数
11366                                       # 自检：107 + 11366 = 11473 ✅
$ grep -i 'http' etc-full/services | head -5
http            80/tcp          www www-http    # WorldWideWeb HTTP
https          443/tcp                          # http protocol over TLS/SSL
```

- **`grep`**：语法 `grep [选项] 模式 文件`；作用：按**整行**匹配。`-c` 只输出条数、`-v` 反向（不含）、`-n` 带行号、`-i` 忽略大小写

### 2. `awk` —— 按列处理

```bash
$ awk -F: '{print $1, $3}' /etc/passwd | head -5
root 0
bin 1
daemon 2
$ awk 'END{print NR}' etc-full/services
11473
```

- **`awk`**：语法 `awk [-F分隔符] '模式{动作}' 文件`；作用：按列取字段（`$1` 第一列、`$NF` 最后一列、`NR` 行号）。默认按空白分列
- **注意**：不给文件参数时它会读标准输入——看起来像"卡住"，`Ctrl+D` 或 `Ctrl+C` 退出

### 3. `sed` / `cut` —— 按行取、按分隔符切

```bash
$ sed -n '1,5p' etc-full/services          # 只打印 1–5 行
$ sed -n '/^ftp/p' etc-full/services | head -3
ftp-data        20/tcp
ftp-data        20/udp
ftp             21/tcp
$ cut -d: -f1 /etc/passwd | head -5        # 按 : 切，取第 1 列（用户名）
root
bin
```

- **`sed`**：语法 `sed [选项] '脚本' 文件`；作用：流式编辑/抽取。`-n` 只输出被 `p` 的行，`s/旧/新/g` 替换
- **`cut`**：语法 `cut -d分隔符 -f列号 文件`；作用：按分隔符切列（比 awk 轻，但不能做条件判断）

### 4. 一行管道做协议统计（含一次踩坑→修正）

```bash
# 第一版（错的）
$ awk -F/ '/^[^#]/{print $2}' etc-full/services | sort | uniq -c | sort -rn | head -3
    60 udp
    47 tcp
    44 tcp          # ← "tcp" 和 "tcp " 被当成两个字符串

# 修正版
$ awk '/^[^#]/{split($2, a, "/"); print a[2]}' etc-full/services | sort | uniq -c | sort -rn | head
   5801 tcp
   5483 udp
     70 sctp
      8 dccp
      4 ddp          # 四项+ddp = 11366，与有效行数一致 ✅
```

- **`sort`**：语法 `sort [选项]`；作用：排序，`-n` 数值、`-r` 倒序、`-k` 指定列
- **`uniq`**：语法 `uniq [-c]`；作用：去重，`-c` 计数。**只合并相邻重复行 → 前面必须先 `sort`**

## 三、实践反馈

| 坑 | 原因 | 正确做法 |
|---|---|---|
| `awk -F/` 取 `$2` 统计协议，出现 `tcp` 与 `tcp ` 两个 | 行尾有注释/别名，切出来带空格 | `split($2, a, "/"); print a[2]` 取纯协议 |
| `uniq -c` 数出来一堆 1 | 没先 `sort`，相同行不相邻 | 永远 `sort | uniq -c` |
| `journalctl` 输出行里字段取错 | 每行有前缀 `月 日 时间 主机 进程:` | `-o cat` 去前缀，或从末尾数 `$(NF-k)` |
| 查 11/15 的日志返回 0 行 | **journald 默认不持久**（`/var/log/journal` 不存在） | 先开持久化，再查历史 |

## 四、数据自检（今天最重要的习惯）

两处都用"加总对得上"来自证没漏数据：

1. `107` 注释行 + `11366` 有效行 = `11473` 总行数 ✅
2. 协议统计 `5801+5483+70+8+4 = 11366` = 有效行数 ✅

> 结论：**统计类命令必须做一次自我校验**，否则切错列、漏匹配都看不出来。

## 五、和前后几天的联系

- Day3 学"怎么找文件/看文件"，今天学"**怎么从文件里抽出要的字段**"
- Day6 把这条管道用在 `sshd` 日志上，直接产出「爆破源 Top5」
