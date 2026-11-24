# Linux Day 1：文件与目录（2026-11-12）

> 环境：VMware + Rocky Linux 8.10，用户 `rocky`；主机名当时为 `cys.rocky.scgs`（Day9 已改为 `cys`）。
> 素材：练习目录 `~/linux-practice`（自建，空目录起步）。
> 来源：`01-linux/命令手账.md` 的「day1 实操」段（当时真实终端输出）。

---

## 一、今天的目标

把 6 个最基础的文件操作命令各用一遍，形成"**先看、再动、动完再确认**"的操作习惯。

## 二、命令与实测

### 1. `pwd` / `ls`

```bash
$ pwd
/home/rocky
$ ls -lah
drwxrwxr-x. 2 rocky rocky   6 Nov 12 09:08 dst
drwxrwxr-x. 2 rocky rocky   6 Nov 12 09:08 src
```

- **`pwd`**：语法 `pwd`；作用：打印当前工作目录的绝对路径（不确定自己在哪时先跑它）
- **`ls`**：语法 `ls [选项] [目录]`；作用：列目录内容。`-l` 长格式（权限/属主/大小/时间）、`-a` 含隐藏文件（`.` 与 `..`）、`-h` 大小人性化、`-t` 按时间

### 2. `mkdir` / `cd`

```bash
$ mkdir -p ~/linux-practice/src ~/linux-practice/dst
$ cd ~/linux-practice && pwd
/home/rocky/linux-practice
```

- **`mkdir`**：语法 `mkdir [-p] 目录`；作用：建目录，`-p` 递归建父目录且已存在不报错
- **`cd`**：语法 `cd [目录]`；作用：切换当前目录（`cd ~` 回家、`cd -` 回上一个目录、`cd ..` 上移一层）
- **`&&`**：前一条成功才执行后一条

### 3. `cp` / `mv` / `rm`

```bash
$ echo "hello linux" > src/note.txt
$ echo "second file" > src/note2.txt
$ cp src/note.txt dst/                  # 复制
$ mv src/note2.txt dst/renamed.txt      # 移动 + 改名
$ ls -lah dst
-rw-rw-r--. 1 rocky rocky 12 Nov 12 09:08 note.txt
-rw-rw-r--. 1 rocky rocky 12 Nov 12 09:08 renamed.txt

$ ls -lah dst/renamed.txt               # 【删前先确认】
$ rm -f dst/renamed.txt
$ ls -lah dst
-rw-rw-r--. 1 rocky rocky 12 Nov 12 09:08 note.txt
```

- **`cp`**：语法 `cp [-r] 源 目标`；作用：复制文件，目录要加 `-r`（`-a` 保留权限/时间戳/软链）
- **`mv`**：语法 `mv 源 目标`；作用：移动文件，**目标不同名时就是改名**（这里 `note2.txt → renamed.txt`）
- **`rm`**：语法 `rm [-f/-r] 目标`；作用：删除，**不可恢复**；`-rf` 递归强删，最危险

## 三、实践反馈

| 观察 | 说明 |
|---|---|
| `ls -lah` 第一列是权限 | `drwxrwxr-x` 里 `d`=目录；后面 9 位是 属主/组/其他人 的读写执行 |
| 同一个 `mv` 既是移动又是改名 | 取决于目标路径是否存在同名 |
| `rm` 没有回收站 | 删之前一定 `ls` 确认（本日先 `ls -lah` 再 `rm -f`） |
| `mkdir -p` 一次建两层 | 省掉"先建父目录"的报错 |

## 四、和后面几天的联系

- Day2 把 `cp` 升级成 `cp -r /etc`，才发现**权限不足会导致"部分复制"**
- Day3 学的 `find/ls` 组合，用来**验证复制是否完整**
- 结论：Day1 的 6 条命令是后面所有操作的地基，重点不是记住参数，而是**"先确认再动手"的条件反射**
