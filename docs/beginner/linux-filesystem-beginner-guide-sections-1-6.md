# Linux 文件系统入门教程（第 1～6 节）

> **适用读者**：零基础或仅有 Windows 使用经验的学习者  
> **目标**：理解 Linux 如何组织磁盘上的数据，并能安全完成日常文件操作、权限设置、挂载与链接  
> **建议环境**：任意 Linux（物理机、虚拟机、WSL2、云服务器均可）  
> **阅读方式**：按节顺序学习；每节末尾有「动手实验」与「自测题」

---

## 目录

1. [第 1 节：文件系统是什么？](#第-1-节文件系统是什么)
2. [第 2 节：路径与目录树（FHS）](#第-2-节路径与目录树fhs)
3. [第 3 节：文件类型与基本操作](#第-3-节文件类型与基本操作)
4. [第 4 节：权限与属主](#第-4-节权限与属主)
5. [第 5 节：磁盘分区与挂载](#第-5-节磁盘分区与挂载)
6. [第 6 节：inode 与硬链接、符号链接](#第-6-节inode-与硬链接符号链接)
7. [附录：命令速查与常见误区](#附录命令速查与常见误区)

---

## 第 1 节：文件系统是什么？

### 1.1 从一个日常命令说起

你在终端执行：

```bash
echo "hello" > notes.txt
cat notes.txt
```

**你感受到的**：有一个叫 `notes.txt` 的文件，里面写着 `hello`。

**硬盘/SSD 上实际存的**：没有叫 `notes.txt` 的“格子”，只有大量固定大小的 **存储块（block）**，每个块是一串 0 和 1。

**文件系统（File System，常缩写 FS）** 是操作系统中的一套规则与数据结构：把块设备上的二进制数据，组织成 **文件、目录、权限、时间戳** 等你熟悉的样子。

```text
┌─────────────────────────────────────────────────────────┐
│  你（和应用）   「我要读 notes.txt」                      │
└───────────────────────────┬─────────────────────────────┘
                            ▼
┌─────────────────────────────────────────────────────────┐
│  文件系统层     查目录 → 找到 inode → 找到数据在哪些块   │
│                （ext4 / XFS / Btrfs 等）                 │
└───────────────────────────┬─────────────────────────────┘
                            ▼
┌─────────────────────────────────────────────────────────┐
│  块设备层       /dev/vdc1 等：按块号读/写物理扇区        │
└─────────────────────────────────────────────────────────┘
```

**口诀**：硬件存 **块**；文件系统管 **名字和结构**；应用程序管 **内容的含义**（文本、图片、数据库记录等）。

---

### 1.2 图书馆类比（三层抽象）

| 层次 | 图书馆类比 | Linux 中大致对应 |
|------|------------|------------------|
| 应用层 | 读者要借《Linux 入门》 | `cat`、`vim`、MySQL、你的程序 |
| 文件系统层 | 图书目录：书名 → 架位 → 书页 | ext4/XFS + 内核 `fs/` 代码 |
| 块设备层 | 仓库里一排排固定大小的货架格子 | 分区如 `/dev/sda1`、`/dev/nvme0n1p2` |

第一节只需建立印象：**你在资源管理器或 `ls` 里看到的“文件名”，是文件系统提供的视图；真正占空间的是数据块。**

---

### 1.3 Linux 的“一切皆文件”

在 Linux 里，**文件（file）** 不只有文本文档：

| 类型 | 说明 | 例子 |
|------|------|------|
| 普通文件 | 字节流 | `notes.txt`、`.jpg`、可执行程序 |
| 目录 | 特殊文件，内容是“名字 → inode”的表 | `/home`、`/etc` |
| 字符/块设备 | 用读写文件的方式访问硬件 | `/dev/null`、`/dev/sda` |
| 管道、socket 等 | 进程通信 | `pipe`、`/var/run/docker.sock` |

好处：内核对外 largely 统一使用 **open / read / write / close**，程序接口简单。

**实例**：向“黑洞”丢弃输出：

```bash
echo "这行不会出现在终端" > /dev/null
echo "这行会" 
```

`/dev/null` 是一个设备文件：写入的数据被丢弃，读取则立即 EOF。

---

### 1.4 文件系统要解决哪些问题？

块设备只提供：**从第 N 块开始，读/写 M 个字节**。文件系统必须回答：

1. **哪些块空闲、哪些已用？**（位图、块组等）
2. **一个文件的内容分散在哪些块？**（连续 extent 或间接块）
3. **文件名 `notes.txt` 对应哪个内部对象？**（目录项 → inode 号）
4. **元数据**：大小、权限、属主、修改时间等
5. **崩溃后如何恢复？**（日志 journal、fsck — 第 7 节及以后深入）
6. **多块盘/网络盘如何接入同一棵树？**（**挂载 mount** — 第 5 节）

---

### 1.5 第一节必会名词

| 名词 | 一句话 |
|------|--------|
| **块（block）** | FS 分配存储的常用单位，常见 4 KiB |
| **分区（partition）** | 一块盘上划出的逻辑区域，如 `/dev/sda1` |
| **格式化（mkfs）** | 在分区上 **新建** FS 结构（通常 **清空** 该分区原有数据） |
| **挂载（mount）** | 把某分区的 FS **接到** 目录树某个目录（挂载点） |
| **根文件系统（rootfs）** | 启动后整棵树的起点，挂载点为 `/` |
| **VFS** | 内核虚拟文件系统，统一 ext4、proc、sysfs 等接口（进阶） |

**易混点**：

- **分区 ≠ 格式化**：分区是“划界”；格式化是在界内“建图书馆索引系统”。
- **`cp` / `mv` ≠ 整盘克隆**：它们是 FS 上的操作；`dd` 是块层复制，层次不同。

---

### 1.6 动手实验（约 10 分钟）

```bash
# 1. 根目录使用哪种文件系统、挂在哪块设备上
findmnt /

# 示例输出（你的设备名可能不同）：
# TARGET SOURCE   FSTYPE OPTIONS
# /      /dev/vdc ext4   rw,relatime

# 2. 当前目录属于哪个文件系统（同 FS 内 mv 大文件才“秒移”）
df -h .

# 3. 块设备与挂载点一览
lsblk -o NAME,SIZE,FSTYPE,MOUNTPOINTS

# 4. 创建文件并查看 inode 号（第 6 节会详解）
echo "hello" > /tmp/notes.txt
ls -li /tmp/notes.txt
# 第一列数字就是 inode，例如：385431 -rw-r--r-- ... notes.txt
```

**观察目标**：

- 出现 `ext4`、`xfs` 等 **类型名**。
- `/`、`/boot` 等可能来自 **不同分区**（若分开挂载）。
- `ls -li` 第一列是 **inode**，同一文件 inode 不变（除非被复制成新文件）。

---

### 1.7 自测题

1. 磁盘上直接存的是“文件名”还是“块里的数据”？  
2. 文件系统位于应用程序和块设备之间的哪一层？  
3. “挂载”把磁盘接到目录树的什么位置？  
4. 格式化和分区，哪一个会创建 inode、空闲块管理等结构？

<details>
<summary>参考答案</summary>

1. 块里的数据；文件名是 FS 元数据。  
2. 中间层。  
3. **挂载点**（一个目录；挂载后该目录“显示”该 FS 的内容）。  
4. **格式化（mkfs）**。

</details>

---

## 第 2 节：路径与目录树（FHS）

### 2.1 单根目录树：没有 C:、D:

Windows 习惯：**盘符** `C:\`、`D:\` 各是一棵树。

Linux 习惯：**只有一棵倒立的树**，根是 **`/`**（读作 root，不是“根目录文件”那个 root 用户）。所有分区、网络盘，都 **挂载** 到这棵树的某个目录上，看起来仍像“一个整体”。

```text
                        /
                       / \
                    /etc  home
                          / \
                       user  shared
                        /
                     documents
                        |
                   notes.txt   ← 路径：/home/user/documents/notes.txt
```

**实例：绝对路径 vs 相对路径**

假设当前目录是 `/home/alice`：

| 写法 | 类型 | 指向 |
|------|------|------|
| `/home/alice/notes.txt` | 绝对路径 | 从根 `/` 开始，唯一确定 |
| `notes.txt` | 相对路径 | `/home/alice/notes.txt` |
| `./documents/x` | 相对路径 | 当前目录下的 documents |
| `../bob/file` | 相对路径 | 上一级目录下的 bob |

特殊目录名：

| 符号 | 含义 |
|------|------|
| `.` | 当前目录 |
| `..` | 父目录 |
| `~` | 当前用户的家目录（shell 展开，如 `/home/alice`） |
| `-` | 部分 shell 中表示 **上一次** 的当前目录（`cd -`） |

```bash
cd /tmp
pwd                    # /tmp
cd /var/log
cd -                   # 回到 /tmp
echo $HOME             # 家目录绝对路径
```

---

### 2.2 路径解析过程（简化）

当你 `cat /home/alice/notes.txt` 时，内核大致：

1. 从 **根 inode** 进入 `/`
2. 在 `/` 的目录项里找 `home` → inode A
3. 在 A 里找 `alice` → inode B
4. 在 B 里找 `notes.txt` → inode C
5. 按 C 记录的数据块位置读取内容

**因此**：路径是一串 **目录项查找**；文件名只存在于 **目录文件** 里，不在 inode 里（重要，第 6 节会再用）。

---

### 2.3 FHS：常见顶层目录做什么用？

**FHS**（Filesystem Hierarchy Standard）约定 Linux 上常见目录用途。发行版略有差异，但大体一致：

| 目录 | 用途 | 实例 |
|------|------|------|
| `/` | 根；整个树的起点 | `findmnt /` |
| `/bin`, `/sbin` | 基本用户/管理员命令（有的系统链到 `/usr`） | `ls`, `ip` |
| `/boot` | 内核与引导相关文件 | `vmlinuz`, `grub` |
| `/dev` | 设备文件 | `/dev/sda`, `/dev/tty` |
| `/etc` | **配置文件**（editable text config） | `/etc/passwd`, `/etc/fstab` |
| `/home` | 普通用户家目录 | `/home/alice` |
| `/lib` | 动态库等 | `libc.so` |
| `/media`, `/mnt` | 可移动介质、临时挂载点 **习惯位置** | U 盘常挂到 `/media/user/XXX` |
| `/opt` | 第三方可选软件 | 商业软件安装目录 |
| `/proc` | 进程与内核信息的 **虚拟 FS** | `cat /proc/cpuinfo` |
| `/root` | **用户 root** 的家目录（不是 `/`） | root 的 `~` |
| `/run` | 运行时状态（重启后重建） | PID 文件、socket |
| `/sys` | 内核与设备信息的虚拟 FS | 电源、块设备属性 |
| `/tmp` | 临时文件（常 world-writable + sticky） | 编译中间文件 |
| `/usr` | 大部分只读用户程序与文档 | `/usr/bin/python3` |
| `/var` | **可变数据**：日志、缓存、数据库 | `/var/log`, `/var/lib/mysql` |

**案例：找日志 vs 找配置**

```bash
# 系统日志（具体路径因发行版而异）
ls /var/log

# 当前机器有哪些用户（配置文件，不是“用户数据文件”）
head -3 /etc/passwd
# root:x:0:0:root:/root:/bin/bash
# ...
```

**案例：`/root` 与 `/` 的区别（初学者常混）**

```bash
# root 用户的家目录
sudo ls -la /root

# 整个系统的根挂载点
ls /
```

---

### 2.4 当前工作目录与 `PATH`

- **`pwd`**：打印当前工作目录（process working directory）。
- **`cd`**：改变当前 shell 的工作目录；子进程会继承。

**可执行文件查找** 与路径无关，由环境变量 **`PATH`** 决定：

```bash
echo $PATH
# /usr/local/bin:/usr/bin:/bin:...

which ls
# /usr/bin/ls

# 当前目录 . 默认通常不在 PATH 里（安全原因）
# 因此要运行当前目录下的脚本：
./myscript.sh
```

---

### 2.5 动手实验

```bash
# 实验 A：绝对路径与 ..
mkdir -p /tmp/fhs-lab/a/b
cd /tmp/fhs-lab/a/b
pwd
cat <<'EOF' > /tmp/fhs-lab/a/b/deep.txt
content
EOF
cat /tmp/fhs-lab/a/b/deep.txt
cat ../../a/b/deep.txt    # 从 /tmp/fhs-lab 相对定位，需先 cd 到合适位置
cd /tmp/fhs-lab
cat ./a/b/deep.txt

# 实验 B：对比 /etc 与 /var
file /etc/passwd
file /var/log 2>/dev/null || ls -ld /var/log

# 实验 C：虚拟文件系统
ls /proc/self/cwd -l    # 符号链接，指向当前 shell 进程的工作目录
readlink /proc/self/cwd
```

---

### 2.6 自测题

1. Linux 路径的起点符号是什么？  
2. `..` 表示什么？  
3. 用户配置文件一般多在 `/etc` 还是 `/var`？  
4. root 用户的家目录路径通常是什么？

<details>
<summary>参考答案</summary>

1. `/`  
2. 父目录  
3. `/etc`  
4. `/root`（不是 `/`）

</details>

---

## 第 3 节：文件类型与基本操作

### 3.1 用 `ls -l` 认文件类型

`ls -l` 第一列第一个字符表示类型：

| 字符 | 类型 |
|------|------|
| `-` | 普通文件 |
| `d` | 目录 |
| `l` | 符号链接（软链接） |
| `b` | 块设备 |
| `c` | 字符设备 |
| `p` | 命名管道 |
| `s` | socket |

**实例**：

```bash
ls -ld /etc /etc/passwd /dev/null
# drwxr-xr-x ... /etc
# -rw-r--r-- ... /etc/passwd
# crw-rw-rw- ... /dev/null
```

**`file` 命令** 看内容/用途（基于 magic，不完全等于内核类型）：

```bash
file /bin/ls /etc/passwd
# /bin/ls: ELF 64-bit ... executable
# /etc/passwd: ASCII text
```

---

### 3.2 创建、复制、移动、删除

| 命令 | 作用 | 本节要点 |
|------|------|----------|
| `touch` | 创建空文件或更新时间戳 | 不创建目录 |
| `mkdir -p` | 创建目录（`-p` 递归） | |
| `cp -a` | 复制 | 默认 **复制数据**；`-a` 尽量保留权限、时间等 |
| `mv` | 移动/重命名 | **同文件系统** 常只改目录项（第 6 节） |
| `rm` | 删除名字 | 删除目录要 `rm -r`； **`rm -rf` 极危险** |

**案例：复制 vs 移动**

```bash
mkdir -p /tmp/op-lab && cd /tmp/op-lab
dd if=/dev/zero of=big.bin bs=1M count=100 status=none   # 100MB 文件

time cp big.bin big.copy.bin
time mv big.bin big.moved.bin   # 同目录“重命名”，几乎瞬间

ls -lh
# big.copy.bin  big.moved.bin  （原 big.bin 已改名为 big.moved.bin）
```

**案例：目录复制**

```bash
mkdir -p src/sub
echo data > src/sub/a.txt
cp -r src dst          # 递归复制整个目录树
diff -r src dst        # 无输出表示内容相同
```

**案例：安全删除习惯**

```bash
# 先看再删
ls -la suspect_dir
rm -i important.conf   # -i 交互确认

# 不要用 root 随便 rm -rf / 或 rm -rf /*
```

---

### 3.3 `cp` 与 `mv` 在文件系统层在干什么？（预习）

| 场景 | `cp` | `mv` |
|------|------|------|
| 同一 FS 内 | 新 inode + 复制所有数据块 | 通常只改目录项（rename） |
| 跨 FS（如 `/` 到 U 盘） | 复制数据 | 内部 ≈ cp + rm 源文件 |

判断 **是否同一文件系统**：

```bash
df --output=source /path/a /path/b
# 若 SOURCE 相同，则同 FS
```

---

### 3.4 重定向与追加

```bash
echo line1 > /tmp/r.txt    # 覆盖写
echo line2 >> /tmp/r.txt   # 追加
cat /tmp/r.txt
```

`>` 由 shell 打开/创建文件再写入；涉及 **权限**（第 4 节）与 **目录是否存在**。

---

### 3.5 查找文件

```bash
# 按名（慢但直观）
find /etc -name 'passwd' 2>/dev/null

# 按类型、大小
find /tmp -type f -size +10M 2>/dev/null

# 最近修改
find /var/log -type f -mtime -1 2>/dev/null | head
```

**案例**：在家目录找大于 50MB 的文件：

```bash
find "$HOME" -type f -size +50M 2>/dev/null
```

---

### 3.6 动手实验

```bash
LAB=/tmp/sec3-lab
rm -rf "$LAB" && mkdir -p "$LAB"/{src,dst}
cd "$LAB"

echo "alpha" > src/a.txt
echo "beta"  > src/b.txt
cp -a src dst/
mv src/b.txt src/b.renamed.txt

find dst -type f
tree "$LAB" 2>/dev/null || find "$LAB"

# 验证 cp 后 inode 不同（复制是新文件）
ls -li src/a.txt dst/src/a.txt
# 两行的 inode 数字应不同
```

---

### 3.7 自测题

1. `ls -l` 里目录第一个字符是什么？  
2. 复制目录树应使用 `cp` 的什么选项？  
3. 如何判断两个路径是否在同一个文件系统上？  
4. `mv` 在同一 FS 内移动大文件为什么通常很快？

<details>
<summary>参考答案</summary>

1. `d`  
2. `-r` 或更完整的 `-a`（归档复制）  
3. `df` 看 SOURCE，或 `stat -f -c %i` / `findmnt` 对比  
4. 通常只更新目录项，不复制数据块（rename）

</details>

---

## 第 4 节：权限与属主

### 4.1 为什么需要权限？

多用户系统上，需要规定：

- 谁可以 **读** 文件内容（r）
- 谁可以 **改** 内容（w）
- 谁可以 **把文件当程序执行**（x，或对目录 **进入/搜索**）

Linux 用 **属主（user）**、**属组（group）**、**其他人（others）** 三组各配 `rwx`。

---

### 4.2 读懂 `ls -l`

```bash
ls -l /etc/passwd
# -rw-r--r-- 1 root root 1234 Jan  1 00:00 /etc/passwd
# │└┬┘└┬┘└┬┘
# │  u  g  o
# 文件类型
```

| 字段 | 含义 |
|------|------|
| `-rw-r--r--` | 类型 + u/g/o 权限 |
| `1` | 硬链接数（第 6 节） |
| `root root` | 属主、属组 |
| 最后 | 大小、时间、名字 |

**目录上的 `x` 特别重要**：没有 **x**，无法 `cd` 进入该目录（即使有 r 列出文件名）。

**案例**：

```bash
mkdir -p /tmp/perm-lab/locked
chmod 644 /tmp/perm-lab/locked    # rw-r--r-- 对目录含义不同，仅作演示
chmod 600 /tmp/perm-lab/locked
# 目录 600：通常 others 不能进；无 x 时自己也难 cd
chmod 755 /tmp/perm-lab
chmod 700 /tmp/perm-lab/locked
ls -ld /tmp/perm-lab /tmp/perm-lab/locked
```

**目录权限速记**：

| 权限 | 对目录的意义 |
|------|----------------|
| r | 可以 `ls` 列出目录项名字 |
| w | 可以在目录里 **创建/删除/改名** 文件（还受父目录等约束） |
| x | 可以 **进入** 目录（`cd`、访问其下路径） |

---

### 4.3 数字权限与 chmod

|r|w|x|二进制|数字|
|---|---|---|-----|-----|
|0|0|0|000|0|
|1|0|0|100|4|
|1|1|0|110|6|
|1|1|1|111|7|

**实例**：

```bash
echo secret > /tmp/p.txt
chmod 640 /tmp/p.txt    # rw-r-----  属主读写，组只读，其他人无
ls -l /tmp/p.txt

chmod u+x /tmp/p.txt    # 属主增加执行位
chmod go-w /tmp/p.txt   # 组和其他去掉写
```

**不要用 `chmod 777` 解决问题**：等于对所有人开放，常见于错误教程。

---

### 4.4 属主与属组：chown、chgrp

```bash
# 需要 root 或 CAP_CHOWN
sudo chown alice:developers /path/to/file
sudo chgrp developers /path/to/file
sudo chown -R alice:developers /path/to/dir   # 递归
```

**`/etc/passwd` 与 `/etc/group`** 定义用户与组（第一节 FHS 中的 `/etc`）。

```bash
id
# uid=1000(alice) gid=1000(alice) groups=1000(alice),27(sudo)

getent group developers
```

---

### 4.5 umask：新建文件的默认权限

创建文件/目录时，权限 = **默认值 & ~umask**（按位）。

```bash
umask          # 常见 0022 或 0002
umask 022      # 设置：新文件 roughly 666&~022=644，目录 777&~022=755
```

**实验**：

```bash
OLD=$LAB  # 或任意可写目录
cd /tmp
umask
touch umask-test-file
mkdir umask-test-dir
ls -ld umask-test-file umask-test-dir
```

---

### 4.6 特殊权限（知道即可）

| 位 | 对文件 | 对目录 |
|----|--------|--------|
| setuid | 执行时以文件属主身份运行 | 基本不用 |
| setgid | 执行时以文件属组身份 | 新建文件继承目录属组 |
| sticky（如 `/tmp` 的 t） | — | 仅属主或 root 能删自己的文件 |

```bash
ls -ld /tmp
# drwxrwxrwt ...  最后的 t 即 sticky
```

---

### 4.7 sudo 与 root

- **root（uid 0）** 在本地 FS 上通常 **绕过** 普通权限检查（仍受 SELinux/AppArmor 等约束）。
- **`sudo`** 以 root 或其他用户身份执行单条命令。

**原则**：日常用普通用户；需要时再 `sudo`。

---

### 4.8 动手实验（完整小场景）

```bash
LAB=/tmp/sec4-lab
sudo rm -rf "$LAB"
sudo mkdir -p "$LAB"/project
sudo chown "$USER:$USER" "$LAB"/project
cd "$LAB"/project

echo "team visible" > readme.txt
chmod 664 readme.txt
mkdir shared
chmod 2775 shared    # setgid：组协作目录常见用法（需组一致才体现）
ls -ld shared readme.txt

# 验证：无读权限时
chmod 000 secret.bin 2>/dev/null; echo x > secret.bin
chmod 600 secret.bin
cat secret.bin         # 属主可读
# 换另一个用户测试（若环境有）: su - otheruser -c 'cat ...' 应失败
```

---

### 4.9 自测题

1. `rwxr-x---` 用数字怎么写？  
2. 目录缺少 **x** 位会导致什么？  
3. umask `022` 时，新建普通文件权限大约是多少？  
4. `/tmp` 上 sticky 位的作用是什么？

<details>
<summary>参考答案</summary>

1. `750`  
2. 无法 `cd` 进入（无法沿路径访问其下文件）  
3. 约 `644`（666 & ~022）  
4. 防止用户互相删除他人文件

</details>

---

## 第 5 节：磁盘分区与挂载

### 5.1 从物理磁盘到你能用的路径

```text
  NVMe / SATA 磁盘
        │
        ▼
  分区表 (GPT/MBR)  →  /dev/nvme0n1p1, /dev/sda2 ...
        │
        ▼
  mkfs 格式化       →  ext4 / xfs / btrfs ...
        │
        ▼
  mount 挂载        →  挂到 /, /home, /mnt/usb ...
        │
        ▼
  应用程序          →  open("/home/alice/file")
```

**实例：查看本机布局**

```bash
lsblk -o NAME,SIZE,FSTYPE,MOUNTPOINTS,UUID

findmnt
# 或
findmnt -D   # 磁盘用法
```

示例（环境不同，输出会变化）：

```text
NAME  SIZE FSTYPE MOUNTPOINTS
vdc   256G ext4   /
```

---

### 5.2 分区 vs 格式化 vs 挂载

| 步骤 | 工具示例 | 作用 | 数据风险 |
|------|----------|------|----------|
| 分区 | `fdisk`, `parted`, `gdisk` | 在磁盘上划分区间 | 操作不当可丢数据 |
| 格式化 | `mkfs.ext4`, `mkfs.xfs` | 建立 FS 超结构 | **通常清空该分区** |
| 挂载 | `mount` | 把 FS 接到目录树 | 不复制用户文件内容 |

**初学者建议在虚拟机或空盘上练习分区**，不要在生产盘上演练。

---

### 5.3 `mount` 与挂载点

**挂载点** 必须是一个 **已存在的目录**。挂载后，该目录原有内容会被 **遮住**（unmount 后恢复可见）。

**案例：挂载 loop 文件（无需真实 U 盘，安全练习）**

```bash
LAB=/tmp/sec5-lab
mkdir -p "$LAB"
cd "$LAB"

# 1. 创建 64MB 空文件充当“小磁盘”
dd if=/dev/zero of=disk.img bs=1M count=64 status=none

# 2. 格式化为 ext4
mkfs.ext4 -F disk.img    # -F 对文件镜像常用

# 3. 挂载
mkdir -p mnt
sudo mount -o loop disk.img mnt
findmnt mnt
# TARGET SOURCE   FSTYPE OPTIONS
# mnt    ...      ext4   rw,relatime

# 4. 像普通目录一样用
echo "mounted here" | sudo tee mnt/hello.txt
cat mnt/hello.txt

# 5. 卸载
sudo umount mnt
ls mnt    # hello.txt 不再可见（卸载后目录可能空）
```

**真实 USB 设备**（设备名务必用 `lsblk` 确认，**写错 dev 会毁盘**）：

```bash
# 仅示意，不要盲抄 sdX
# sudo mount /dev/sdb1 /mnt/usb
# sudo umount /mnt/usb
```

---

### 5.4 `/etc/fstab`：开机自动挂载

`/etc/fstab` 每行一条：**什么设备 → 挂到哪 → 什么类型 → 什么选项**。

```bash
cat /etc/fstab
# 典型字段：
# UUID=...  /  ext4  defaults  0  1
```

**为什么用 UUID？** 设备名 `/dev/sda1` 可能因插拔顺序变化；UUID 更稳定。

```bash
blkid
lsblk -o NAME,UUID,FSTYPE,MOUNTPOINTS
```

**修改 fstab 前**：先用 `mount` 命令测试成功，再写入；错误 fstab 可能导致启动进 emergency mode。

---

### 5.5 挂载选项（常见）

| 选项 | 含义 |
|------|------|
| `rw` / `ro` | 读写 / 只读 |
| `noexec` | 禁止执行该 FS 上的程序 |
| `nosuid` | 忽略 setuid 位 |
| `nodev` | 不解释设备文件 |
| `relatime` | 相对时间更新 atime（性能） |
| `bind` | 把已有目录再挂到别处（高级） |

查看当前挂载选项：

```bash
findmnt -o TARGET,OPTIONS /
mount | grep '^/dev'
```

---

### 5.6 跨文件系统的 `mv` 为何变慢？

```bash
df --output=source,target / /mnt/usb
# 若 SOURCE 不同，mv 大文件 = 复制 + 删除源
```

**案例（逻辑说明）**：

```text
/home/user/big.iso  在 /dev/vdc (ext4)
/mnt/usb/           在 /dev/sdb1 (vfat)

mv /home/user/big.iso /mnt/usb/
  → 内核无法 rename 跨设备
  → 用户态/内核复制全部数据到 U 盘，再 unlink 源文件
```

---

### 5.7 动手实验 checklist

在 `/tmp/sec5-lab` 完成 loop 挂载实验（见 5.3）后，追加：

```bash
# 查看镜像 UUID
sudo blkid "$LAB/disk.img"

# 写入 fstab 行（练习用，可选，需 root；完成后建议删除该行）
# UUID=xxxx  /tmp/sec5-lab/mnt  ext4  loop,defaults  0  0
```

---

### 5.8 自测题

1. 挂载点是什么？  
2. 为什么 fstab 推荐 UUID 而不是 `/dev/sda1`？  
3. `mkfs.ext4` 对分区做了什么层次的事？  
4. 卸载命令是什么？卸载前为什么要确保没有进程在用该目录？

<details>
<summary>参考答案</summary>

1. 目录树上接入某个 FS 的目录位置。  
2. 设备名可能随启动顺序改变，UUID 稳定。  
3. 在已有分区上 **创建** ext4 文件系统结构（格式化）。  
4. `umount`； busy 时卸载会失败或导致数据不一致，需 `cd` 离开或 `lsof` 查占用。

</details>

---

## 第 6 节：inode 与硬链接、符号链接

### 6.1 inode 是什么？

**inode（index node）** 是文件系统里描述 **一个文件对象** 的内核/磁盘结构，通常包含：

- 文件类型、权限、属主属组
- 大小、时间戳
- **数据块指针**（内容在哪些 block/extent）
- 链接计数（硬链接数）

**不包含**：文件名（文件名在 **目录** 里）。

```text
  目录文件 "project/"
  ┌──────────────────────────────┐
  │  readme.txt  →  inode 12345  │
  │  main.c      →  inode 12346  │
  └──────────────────────────────┘

  inode 12345
  ┌──────────────────────────────┐
  │  mode, uid, gid, size, ...   │
  │  blocks → [LBA ...]          │
  └──────────────────────────────┘
```

---

### 6.2 查看 inode

```bash
ls -li file.txt
# 385431 -rw-r--r-- ... file.txt

stat file.txt
# Inode: 385431
# Links: 1
# ...
```

**实例输出含义**（数字随机器变化）：

```text
  File: notes.txt
  Size: 6
  Device: 254,32
  Inode: 385431
  Links: 1
```

同一 **Device** + 同一 **Inode** = 同一个文件对象。

---

### 6.3 硬链接（hard link）

**硬链接**：在 **另一个目录** 里增加一条 **目录项**，指向 **同一个 inode**。

```bash
LAB=/tmp/sec6-lab
mkdir -p "$LAB" && cd "$LAB"
echo "same data" > original.txt
ln original.txt hardlink.txt
ls -li original.txt hardlink.txt
# inode 相同，Links: 2

stat -c '%i %n %h' original.txt hardlink.txt
# 相同 inode 号，链接计数为 2

rm original.txt
cat hardlink.txt          # 仍可读，数据仍在
# 直到链接计数为 0，inode 对应的数据块才会被释放
```

**限制**：

1. **不能跨文件系统**（inode 只在当前 FS 内有意义）
2. **不能对目录** 随意创建硬链接（避免破坏目录树结构；`.`/`..` 除外，由系统维护）

---

### 6.4 符号链接（symbolic link / soft link）

**符号链接**：一种 **特殊文件**，内容是 **另一个路径字符串**。

```bash
ln -s hardlink.txt symlink.txt
ls -li symlink.txt
# 不同 inode；第一列显示 -> hardlink.txt

readlink symlink.txt
# hardlink.txt

rm hardlink.txt
cat symlink.txt           # 报错：断链 dangling symlink
```

| 对比 | 硬链接 | 符号链接 |
|------|--------|----------|
| 本质 | 同一 inode 的别名 | 存路径的小文件 |
| 跨 FS | 不可以 | 可以 |
| 指向目录 | 一般不允许用户创建 | 允许（谨慎使用） |
| 源删后 | 仍可用 | 断链 |
| `ls -li` | inode 相同 | inode 不同，有 `->` |

**案例：常用相对软链**

```bash
mkdir -p "$LAB"/var/log
ln -s ../var/log "$LAB"/logs
ls -l "$LAB"/logs
# logs -> ../var/log
```

---

### 6.5 同一文件系统内 `mv` 为何快？（完整解释）

**rename 系统调用**（同 FS）大致做：

- 在目标目录增加目录项（或改名字）
- 在源目录删除目录项
- **inode 与数据块不动**

因此 100GB 文件 **同盘 mv** 也几乎是瞬间。

**跨 FS**：无法只改 inode 引用（目标 FS 有自己的 inode 空间），必须 **复制数据块** + 删除源 — 与 `cp` 同级慢。

**验证**：

```bash
df --output=source /tmp /var/tmp
# 若相同
time mv /tmp/bigfile /var/tmp/   # 应很快

# 若不同挂载点（如 / 与 /mnt/usb）
time mv /tmp/bigfile /mnt/usb/   # 慢
```

---

### 6.6 `cp` 与 inode

`cp` 创建 **新 inode**，复制 **数据**（除非 reflink 等优化，进阶）：

```bash
cd "$LAB"
echo x > a.txt
cp a.txt b.txt
ls -li a.txt b.txt
# inode 不同；内容相同
```

---

### 6.7 目录与链接计数

每个目录至少 **2** 个硬链接计数来源：`.` 与父目录中的 `dirname` 项；子目录还有 `..`。

```bash
mkdir sub
ls -ld sub
# 链接数 often 2；mkdir 后父目录链接数 +1
```

---

### 6.8 动手实验（综合）

```bash
LAB=/tmp/sec6-lab
rm -rf "$LAB" && mkdir -p "$LAB" && cd "$LAB"

echo "inode demo" > f1
ln f1 f1-hard
ln -s f1 f1-soft
ls -li f1 f1-hard f1-soft

# 同 FS rename 速度（小文件仅演示命令）
mv f1 f1-renamed
ls -li f1-hard f1-renamed f1-soft
cat f1-hard
# soft link 若用相对路径创建，可能需 ln -sf 重建

# 断链演示
rm f1-renamed
file f1-soft
cat f1-soft 2>&1
```

---

### 6.9 自测题

1. 文件名存在 inode 里吗？  
2. 硬链接与源文件 inode 相同还是不同？  
3. 为什么硬链接不能跨文件系统？  
4. 删除文件在 FS 层实际常做什么？（提示：链接计数）  
5. 同 FS 内 `mv` 大文件快的原因一句话说明。

<details>
<summary>参考答案</summary>

1. 不在；在目录项里。  
2. 相同。  
3. inode 只在单个 FS 内编号与管理。  
4. 删除目录项，inode 链接计数减 1；为 0 时释放数据块。  
5. 只改目录项（rename），不复制数据块。

</details>

---

## 附录：命令速查与常见误区

### A. 命令速查

| 主题 | 命令 |
|------|------|
| 路径/挂载 | `pwd`, `cd`, `findmnt`, `df`, `lsblk`, `mount`, `umount` |
| 列表/类型 | `ls -l`, `ls -li`, `file`, `stat` |
| 操作 | `cp -a`, `mv`, `rm`, `mkdir -p`, `touch` |
| 权限 | `chmod`, `chown`, `chgrp`, `umask` |
| 链接 | `ln`, `ln -s`, `readlink` |
| 查找 | `find`, `which` |

### B. 常见误区

| 误区 | 正解 |
|------|------|
| “删除文件就是清空磁盘那些块” | 先减链接计数；可能仍有硬链接；块延迟回收 |
| “mv 总是比 cp 快” | 仅 **同文件系统** 的 mv 通常只改元数据 |
| “chmod 777 最省事” | 安全隐患；应最小权限 |
| “/ 就是 root 用户的家” | `/` 是根挂载点；root 家是 `/root` |
| “格式化只是清空空文件” | mkfs 重建 FS 结构，数据通常不可恢复 |

### C. 后续学习（大纲第 7～9 节预告）

- **第 7 节**：ext4、XFS、Btrfs 特性与选型  
- **第 8 节**：journal、页缓存、`sync`、掉电与 fsck  
- **第 9 节**：VFS、从 `read()` 到块层的 I/O 路径  

---

*文档版本：合并第 1～6 节 · 适用于通用 Linux 发行版（Debian/Ubuntu/RHEL/Fedora 等）*
