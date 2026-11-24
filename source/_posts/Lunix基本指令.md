---
title: Lunix基本指令
date: 2026-11-22 15:41:47
tags: [Linux, 命令行, 权限, 包管理, shell]
description: 'Linux 入门笔记（2025-05-18 ~ 07-22 六次课）：Linux 发展史与版本、基本指令全套（pwd/ls/cd/stat/rm/cp/mv/cat/重定向/管道/date/find/tar）、绝对与相对路径、文件类型不由后缀决定、shell 外壳的本质、文件权限与 umask、粘滞位、yum 与 apt 及包管理器工作原理'
---

> 内容来源：`D:\computer source\Lunix\116_117-master\板书`（都卡片子截图，2025 年 5 月 18 日 ~ 7 月 22 日共 6 次课）
>
> - `20250518_Linux初识`：课程地图与学习路线、计算机与操作系统发展史、Unix → Linux、开源与闭源、技术版本与商业版本、应用场景、环境安装与云服务器、xshell 远程登录、`adduser`/`passwd`/`userdel`
> - `20250525_基本指令1`：`pwd`/`whoami`/`ls`/`clear`/`mkdir`/`cd`/`touch`、`ls -l`/`ls -a`、目录树与路径、`stat`、`mkdir -p`、`rm -r`/`rm -rf`、绝对路径与相对路径、执行程序的路径问题、文件类型与 `file`
> - `20250525_基本指令2`：`man` 手册九个 section、`cp`/`mv`、`cat`/`more`/`less`/`head`/`tail`、`echo`/`printf`、输出重定向与追加重定向、输入重定向、管道、`date` 格式化
> - `20250715_外壳理解`：广义/狭义操作系统、用户 → shell → 内核、bash 是外壳程序、王婆与小李小张的故事、`find`/`which`、指令本质是可执行文件、打包压缩与 `scp`、`uname -r`、热键
> - `2025_07_19_权限`：权限 = 人 + 文件属性、root 与普通用户、`su`、三种角色、权限串解读、`chmod`/`chown`/`chgrp`、`sudo` 与 `/etc/sudoers`、块设备与字符设备
> - `2025-07-22_yum_apt`：目录的 r/w/x、起始权限 666/777 与 `umask`、粘滞位 `t`、安装软件的三种方式、yum 与 apt、包管理器与镜像站、依赖列表
>
> 说明：这 6 张超长截图的后半段均为空白留白，有效内容集中在每张图的前几屏，本篇按"有效板书"整理。

## 目录

- **一、Linux 初识：它是什么，我们为什么要用它**
- **二、基本指令（一）：文件与目录操作**
- **三、基本指令（二）：查看、重定向与管道**
- **四、外壳理解：shell 到底是什么**
- **五、文件权限**
- **六、目录权限、umask、粘滞位与包管理**
- **七、命令速查表**

---

## 一、Linux 初识：它是什么，我们为什么要用它

### 1.1 课程地图与学习路线

```text
1. Linux初识、Linux环境、Linux命令、Linux开发环境      —— 1.0
2. Linux系统                                          —— 2.0
3. Linux网络                                          —— 3.0
4. MySQL                                              —— 4.0
```

两条主线的对照（箭头串联）：

```text
20 学时：CentOS 7 + C   + vim/gcc/g++/gdb
60 学时：Ubuntu 20.04 + C++ + vscode
        ↓
命令行为主，距离操作系统最近
        ↓
    Linux 操作系统
```

> 关键认知：**Linux 学习要以命令行为主**——命令行是"距离操作系统最近"的入口。

### 1.2 计算机与操作系统的发展史

时间线：

```text
1946.2.14 埃尼阿克（ENIAC） → 1945 ... 1991 → 美苏冷战 → 军备竞赛 → 新的科技
                                                        ↓
                          "计算机技术，就是再这样的时代背景下诞生的！"
                  军用需求 → 民用技术 → 转民用 → 计算机被我们广泛使用
```

发展脉络（以"埃尼阿克"为起点的分叉网络）：

- **大型机阶段**：计算机只能被企业使用 → 大型计算机 → 银行、金融机构、高校、研究机构
- **小型化**：半导体技术 → 计算机小型化（过程）；摩尔定律（过程）
- **个人计算机**：乔布斯 → 苹果 → 个人计算机
- **产业化路径**：军工投入 → 技术 → 硅谷模式 → 「技术 + 人 + 钱」 → 孵化大公司 → 华尔街 → `IBM`、`进入高校！！`

**计算机制造商 = 一整套生态**（板书大方框）：

```text
微软（软件公司）—— 比尔盖茨   → 操作系统
IBM
英特尔                        → 芯片
内存 / 磁盘 / 显卡 / 显示器 / 网卡 ...
                        ↓
               孵化出计算机的制造商 → 生态
```

> 结论：一台计算机从来不是"一家公司做出来的"，而是**一整套分工明确的生态**。

### 1.3 操作系统的诞生：Unix → Linux

```text
操作系统诞生 → 收费的 → Unix（肯·汤普森）
                        ↓ 汇编语言 → 进入高校！
```

Unix 分叉出的三条支线：

| 支线 | 板书批注 | 走向 |
| --- | --- | --- |
| `微软 --- windows` | 微软立马跟进 | → windows 图形界面的操作系统 |
| `苹果 --- macos` | 图形化界面的开发团队 | → **施乐公司**：自己的实验室有一个带图形化界面的操作系统 |
| `类 unix` | — | （Linux、BSD 等） |

**重要提醒（板书绿框强调）**：**所有的操作系统，全部都是黑屏命令行的！**（图形界面只是外面加的一层）

Unix → Linux 的交接：

- 丹尼斯·里奇：**Unix 操作系统用 C 语言进行重写！**
- 学生接触到 Unix → **1991 年，赫尔辛基大学，雷纳斯·托瓦兹（Torvalds）接触 Unix 系统**
- 模仿 → 开发自己的操作系统 → **1.1w → 开源！**
- 社区画像（并列批注）：黑客、工程师、"讨厌苹果微软，讨厌收费的顶级工程师"、"免费在给 Linux 社区写代码？？"

```text
Linux 操作系统
Linux Is Not Unix        ← 名字来源：Linus + Unix
Latest Release 6.14.7
```

### 1.4 总结：开源与闭源

```text
Linux 操作系统是一个开源的操作系统
        ↓
可以以最小的成本直接获取源代码
```

| | 开源（Linux） | 闭源 |
| --- | --- | --- |
| 效率 | 高 | 低 |
| 稳定 | 稳定 | 不稳定 |
| 安全 | >= 安全 | 一般 |
| bug 处理 | 发现、更新、维护、解决 bug 的效率更高！ | 依赖厂商 |

板书圈注的三个关键词：**高效 → 稳定 → 安全**。

**老板视角**：`你是老板：Linux → 免费`。

几个必须想清楚的问题（板书追问）：

- `安全` → `Linux 有病毒吗?` → **生态问题**（用的人少，攻击收益低；不是"技术上不能中毒"）
- `是不是 Linux 操作系统的版本越新越好呢` → **不是的！**（稳定优先于新）
- `Linux 操作系统，你应该如何给老板说？` → **生态适合我们！**

### 1.5 Linux 版本：技术版本与商业版本

#### （1）技术版本 = 内核版本

```text
内核版本号规则：主版本号 . 次版本号 . 修正次数
      偶数：稳定版本
      奇数：测试版本
例：6.14.7
```

实测 `uname -a`：

```bash
[sh@bite-alicloud ~]$ uname -a
Linux bite-alicloud 3.10.0-1160.119.1.el7.x86_64 #1 SMP Tue Jun 4 14:34:51 UTC 2024 x86_64 x86_64 x86_64 GNU/Linux
```

> 旁注疑问（板书）：`Linux 社区很活跃吗?`

#### （2）商业版本 = 发行版

```text
ubuntu   CentOS
debian   slackware   redhat(linux)
arch     SUSE        fedora
```

**生态与人群关系**（右侧大图）：

```text
一定能够让更多人使用！
→ 增加 OS 客户群体，增加自己操作系统的影响力

学生群体   ── 论坛 →（学生关注、大数据关注的软件配套）─┐
企业工程师 ── 论坛 →（工程师配套的软件）──────────────┤
黑客       ── 论坛 ─────────────────────────────────┼→ Linux 内核 → Linux 社区 →【生态！】
其他       ── 论坛 →（攻防、安全 ...）────────────────┘
```

> **"人群 → 论坛 → 内核"** 是这张图的核心：生态的本质是"不同人群各自需要的软件，都能在内核上找到"。

### 1.6 Linux 应用场景

- **服务器领域**（最主流）：Linux 服务器操作系统在整个服务器操作系统市场格局中占据了越来越多的市场份额，已经形成了大规模市场应用的局面；凭借开源、安全、稳定等特性，获得了电信、交通、电子商务等关键领域的青睐；此外凭借**低功耗**与**高稳定性**的优势……
- **桌面领域**：近年来 Linux 在个人桌面领域的市场份额仍较小。
- **嵌入式领域**：低成本、强大功能、良好移植性，使 Linux 在嵌入式系统中广泛应用——手机、平板电脑、路由器、电视、电子游戏机。**Android 就是基于 Linux 内核**。
  - 板书旁链：`手机厂商的操作系统 → 安卓 → Linux`，批注：`设置 → 系统信息 → 内核版本`（可以在自己的安卓手机上查看内核版本）
- **云计算 / 大数据领域**：据 Linux 基金会的研究，**80% 的企业已经使用 Linux 操作系统进行云平台部署**；Linux 已成为最受欢迎的云计算/大数据平台。

> 一句话：**未来是我们程序的服务端的平台！！！**

### 1.7 环境安装与云服务器

| 方式 | 评价 |
| --- | --- |
| 双系统 | **不推荐**（切换麻烦、装坏风险高） |
| WSL | 可用，Windows 下体验较好 |
| 安装虚拟机 | **推荐**（最便宜的隔离方案） |
| **云服务器** | **最常用的开发方式** |

云服务器流程：

1. 登录云厂商官网（如腾讯云）**注册**
2. **购买**（挑最便宜的学生机即可）
3. **安装 CentOS 7.X**
4. 控制台 → 记录公网 IP →（群里会发 `xshell` 安装包）
5. **重置密码**
   - 账号：`root`（选择）
   - 密码：自己设置密码

#### xshell 远程登录

```text
[xshell（装在 windows 里）]  ⇄ 远程登录 ⇄  [远程 Linux]
```

会话设置弹窗：

```text
名称(N):    （自定义）
协议(S):    SSH
主机(H):    120.24.7?.129     ← 云服务器公网 IP
端口(U):    22                ← SSH 默认端口
用户身份验证
  方法(M):  Password
  用户名(U): root
  密码(P):   ******
[完成]  [取消]  [帮助]
```

> 端口 `22` 是 SSH 的默认端口，这也是后面学网络时会遇到的第一个"知名端口"。

#### 用户管理三连

```bash
[root@bite-alicloud ~]# adduser wangwu
[root@bite-alicloud ~]# passwd wangwu
Changing password for user wangwu.
New password:                 ← 密码不回显（输入时不显示任何字符）
Retype new password:
passwd: all authentication tokens updated successfully.
```

```bash
[root@bite-alicloud ~]# userdel -r wangwu
```

- `[root@bite-alicloud ~]` ↦ **Linux 的管理员**
- `userdel -r` ↦ **删除用户（连带家目录）**
- 板书结论（绿色）：**Linux 系统是一款多用户的操作系统！**

### 1.8 结尾预告：常见指令

- **先快速讲 6~8 个指令**
- **后面，一个一个地给大家说明详细**

---

## 二、基本指令（一）：文件与目录操作

### 2.1 八条最先要熟的指令

```bash
[whb@bite-alicloud ~]$ pwd            # 我当前在哪个路径下
/home/whb
[root@bite-alicloud ~]# pwd
/root

[whb@bite-alicloud ~]$ whoami         # 谁在访问 Linux 系统
whb
[root@bite-alicloud ~]# whoami
root

[root@bite-alicloud ~]# ls            # 罗列当前目录下的文件名
113  113backup  114  115  116_117  test  xjh

[root@bite-alicloud ~]# mkdir 116_117   # 在当前目录下新建一个目录/文件夹
[root@bite-alicloud ~]# ls
113  113backup  114  115  116_117  test  xjh

[root@bite-alicloud ~]# cd 116_117      # 更改路径
[root@bite-alicloud 116_117]# touch code.c   # 新建一个普通文件
```

> 两个概念先立住：
> 1. **windows 系统中，标识文件夹唯一性，是通过路径标识的！**
> 2. **无论是 windows 还是 Linux，我们登录成功都会默认处在一个默认的路径下！**
>
> 另外：`文件类 == 目录`（板书写法，即"文件"与"目录"都属于文件系统的对象）。

`clear`：清屏操作。

### 2.2 `ls` 详解

```bash
[root@bite-alicloud 116_117]# ls -l        # 显示更多文件属性
total 0
-rw-r--r-- 1 root root 0 May 25 14:58 code.c

[root@bite-alicloud 116_117]# ls -l -a     # 显示当前路径下的隐藏文件或者文件夹
```

**理论（板书的一个好问题）**：

> 问：如果我在 Windows 新建一个空的文件，这个文件要不要占据磁盘空间？
> **文件的属性 是不是 数据？? 是的！！**

```text
文件 = 文件的内容 + 文件的属性
        ↓              ↓
   对内容做操作      对属性做操作
                   （属性也是数据）
```

- **Linux 系统中，可以存在隐藏文件，隐藏文件往往以 `.` 开头**
- `.` 通常表示当前路径，`..` 通常表示上级路径
- `.` 当前路径的意义：**执行当前路径下的程序**（见 2.5）

### 2.3 目录树与路径

```bash
[root@bite-alicloud /]# pwd
/
```

```text
                    /            ← Linux 根目录
   ┌────┬────┬────┬──┴─┬────┬────┬────┐
  bin  boot  dev  etc  home  lib  root ...
                      ┌─┴──┐
                    whb   ...
                     └ hello.txt
```

- **Linux 的整个文件结构，是树状结构！**
- **多叉树**：任何一个节点，都只有一个父节点
- **根目录不能再前进了**（`cd ..` 到 `/` 就到头）
- **结论 1**：多叉树，叶子节点一定是普通文件或空目录
- **结论 2**：路上节点（非叶子节点）本质就是文件夹或者目录

**为什么标识文件唯一性靠路径？**

> 任何一个多叉树节点，都只有一个父节点；从目标文件定位的时候，**路径就是唯一的**！
> `/usr/bin/hello.txt` —— 唯一的！**绝对路径！**

### 2.4 `cd` 与路径切换

```bash
[root@bite-alicloud ~]# cd ..              # 切换到上一级

[root@bite-alicloud ~]# cd 绝对路径          # 切换到目标路径下
[root@bite-alicloud ~]# ls /root/116_117/code.c
/root/116_117/code.c
[root@bite-alicloud ~]# pwd
/root/116_117
[root@bite-alicloud ~]# ls ../code.c
../code.c

[root@bite-alicloud ~]# pwd
/root/116_117/dira
[root@bite-alicloud dira]# ls ../../code.c
../../code.c

[root@bite-alicloud ~]# cd -
/root/116_117                              # 回到上次所处的路径

[root@bite-alicloud ~]# cd ~
[root@bite-alicloud ~]# pwd
/root                                      # 当前用户的家目录
```

```bash
[root@bite-alicloud ~]# cd ~
[root@bite-alicloud ~]# pwd
/root                                      # root 账号的家目录

[whb@bite-alicloud ~]$ pwd
/home/whb                                  # 普通用户的家目录 = /home/XXX
```

> **`~` 是"当前用户的家目录"的简写**：root 是 `/root`，普通用户是 `/home/用户名`。所以同一个 `cd ~`，不同用户去到的位置不同。

### 2.5 绝对路径 vs 相对路径

| | 写法 | 参照点 | 特点 |
| --- | --- | --- | --- |
| **绝对路径** | `/usr/bin/hello.txt` | 从 `/` 作为参照点 | **唯一**，与当前位置无关 |
| **相对路径** | `./bin/hello.txt` | 自己当前所在目录 | 简洁，但**换目录就可能失效** |

> - 相对路径默认标识：是从我自己当前的目录为参照点的！
> - 一旦我所在的目录发生变化，**相对路径就可能会失效**！
> - 对于确定的绝对路径，相对路径就不能保证唯一性。

**最佳实践**：

1. **配置文件**里要配置各种路径 → **建议用绝对路径**
2. 否则，常规操作一般用**相对路径**（短、好打）

### 2.6 `mkdir -p` 与 `rm`

```bash
[root@bite-alicloud 116_117]# mkdir -p a/b/c/d      # 创建一串路径
```

```bash
[root@bite-alicloud 116_117]# rm -r a
rm: descend into directory 'a'? y
rm: descend into directory 'a/b'? y
rm: descend into directory 'a/b/c'? y
rm: remove directory 'a/b/c'? y
rm: remove directory 'a/b'? y
rm: remove directory 'a'? y

[root@bite-alicloud 116_117]# rm -rf a              # 直接删，不问
```

- `rm` 删文件
- `rm -r` 删目录（递归，逐个询问）
- `rm -rf` 删目录（递归，不询问）——**危险命令，`rm -rf /` 是经典事故**

> 如果你的操作出现了退不出的情况，快捷方式 **`ctrl + c` 终止正在做的操作**。

### 2.7 执行一个程序：为什么要 `./`

```bash
-rwxr-xr-x 1 root root 8360 May 25 17:05 my.exe     ← 已经在夹子里，且有 x 权限

[root@bite-alicloud 116_117]# ./my.exe
hello Linux
[root@bite-alicloud 116_117]# pwd
/root/116_117
[root@bite-alicloud 116_117]# /root/116_117/my.exe
hello Linux

[root@bite-alicloud 116_117]# my.exe
-bash: my.exe: command not found
```

> **你要执行一个程序的前提，就是要先找到它！**
> - 是文件吗？ → 是
> - **路径找到了吗？** → 没有！
>
> `my.exe` 会被 shell 当成"命令名"去 `PATH` 里找，而当前目录默认不在 `PATH` 里；所以要写 `./my.exe`（明确指定"当前目录下的这个文件"）或写绝对路径。

### 2.8 文件类型：不由后缀决定

> **Linux 的文件类型，不由文件后缀决定。**
> 但这**不代表 Linux 不用后缀**（后缀是给人看的、也是给编译器等工具识别用的）。

```bash
[root@bite-alicloud 116_117]# ls -l
total 44
drwxr-xr-x 3 root root 4096 May 25 17:31 a
-rw-r--r-- 1 root root   78 May 25 17:05 code.c
drwxr-xr-x 3 root root 4096 May 25 15:23 dira
drwxr-xr-x 2 root root 4096 May 25 15:29 dirb
drwxr-xr-x 2 root root 4096 May 25 16:27 dirc
-rw-r--r-- 1 root root    0 May 25 17:28 hello.txt
-rwxr-xr-x 1 root root 8360 May 25 17:05 my.exe
-rwxr-xr-x 1 root root 8360 May 25 17:33 world.txt
```

**`file` 命令直接看真实类型**：

```bash
[root@bite-alicloud 116_117]# file a
a: directory
[root@bite-alicloud 116_117]# file code.c
code.c: C source, ASCII text
[root@bite-alicloud 116_117]# file my.exe
my.exe: ELF 64-bit LSB executable, x86-64, version 1 (SYSV), 71539b8f4e0a1e0c619a62f4c7f2167493, not stripped
[root@bite-alicloud 116_117]# file hello.txt
hello.txt: empty
```

> 注意 `world.txt` 和 `my.exe` **大小完全一样（8360）**、权限也一样（`-rwxr-xr-x`）——因为 `world.txt` 就是把 `my.exe` 复制过去改了名字。**后缀变了，类型没变**。

**后缀骗人的后果**：

```bash
[root@bite-alicloud 116_117]# cat my.txt
#include <stdio.h>

int main()
{
    printf("hello Linux\n");
    return 0;
}
[root@bite-alicloud 116_117]# gcc my.txt
my.txt: file not recognized: File format not recognized
collect2: error: ld returned 1 exit status
```

> `gcc` 靠**后缀**决定怎么处理文件：`.txt` 它不认识 → 不编译，直接当链接输入 → 报"文件格式无法识别"。**所以后缀不决定"类型"，但决定"工具怎么处理它"。**

### 2.9 `stat`：看文件属性的全貌

```bash
[root@bite-alicloud 116_117]# stat hello.txt
  File: 'hello.txt'
  Size: 0               Blocks: 0          IO Block: 4096   regular empty file
Device: fd01h/64769d   Inode: 917771      Links: 1
Access: (0644/-rw-r--r--)  Uid: (    0/    root)   Gid: (    0/    root)
Access: 2025-05-25 17:26:01.080773897 +0800
Modify: 2025-05-25 17:26:01.080773897 +0800
Change: 2025-05-25 17:26:01.080773897 +0800
```

三个时间要分清：

| 字段 | 含义 |
| --- | --- |
| `Access` | 最后访问时间（读一次就变） |
| `Modify` | 最后修改**内容**的时间 |
| `Change` | 最后修改**属性**（权限/属主等）的时间 |

> 另外还能看到 `Inode`（文件在磁盘上的唯一编号）与 `Links`（硬链接数）——这两个是后面"文件系统"课程的主角。

### 2.10 `tree`：一眼看清目录结构

```bash
# 你确认还没安装 root
# centos:
yum install -y tree
# ubuntu:
apt install -y tree
```

> 这里已经暗示了后面 7-22 的内容：**不同发行版的包管理器不同**（yum / apt）。

---

## 三、基本指令（二）：查看、重定向与管道

### 3.1 `man`：手册的九个 section

```text
1  Executable programs or shell commands
2  System calls (functions provided by the kernel)
3  Library calls (functions within program libraries)
4  Special files (usually found in /dev)
5  File formats and conventions eg /etc/passwd
6  Games
7  Miscellaneous (including macro packages and conventions), e.g. man(7), groff(7)
8  System administration commands (usually only for root)
9  Kernel routines [Non standard]
```

> 板书圈注了 **1、3、2**：先会用 1（命令）、3（库函数）、2（系统调用）就够了。
> 记忆法：**1 命令、2 系统调用、3 库函数、5 配置文件格式、8 管理员命令**。

### 3.2 `cp`：拷贝

```bash
cp src dst                 # 拷贝文件
cp -r lesson4 lesson4-backup   # 拷贝目录（要 -r 递归）
```

板书标注的两个分支：

- `cp` → **拷贝目录 / 剪切拷贝**（拷贝文件）
- `cp -r` → **指定目录**

### 3.3 `mv`：移动 = 重命名

```bash
mv src dst
```

```text
文件名 hello.txt → mv hello.txt world.txt   ← 重命名（同一目录内"移动"就是改名）
/a/b/c/hello.txt → mv hello.txt /a/b/c/     ← 移动到指定目录
```

> **`mv` 同时承担"剪切"和"重命名"两件事**——因为它做的动作就是"把路径换个位置"，同一目录下换个名字本质一样。（对比 `cp` + `rm`，`mv` 是原子且不产生副本的。）

### 3.4 `cat` / `more` / `less` / `head` / `tail`

```bash
cat 文件名            # 打印文件的内容！
```

- 用途：**查看小文件的内容**
- **内容过长，就会导致刷屏**（一个屏幕滚不完）

```bash
more or less（推荐）
less 文件名
```

`less` 里的操作：

```text
上下键        —— 翻动
/搜索关键字   回车 —— 搜索，n 跳到下一个
```

> **日志文件就用 `less`！**（日志动辄几十万行，`cat` 只会刷屏）

```bash
head 文件名        # 看前面若干行（默认 10 行）
tail 文件名        # 看后面若干行（默认 10 行）
```

```bash
[root@bite-alicloud lesson4]# head -5011 hello.txt > temp.txt
[root@bite-alicloud lesson4]# tail -11 temp.txt
hello 5000
hello 5001
hello 5002
...
```

> 这个例子的用意：`head -n` / `tail -n` 的 `n` 可以任意指定；把 `head` 的结果**重定向**到文件，再用 `tail` 看末尾——就是"重定向"最直观的用法。

### 3.5 补充理论：Linux 下"一切皆文件"

```text
Linux系统下一切"文件"

写入 → 文件 → 向显示器文件进行写入
 1. 向显示器打印
 2. 从键盘 scanf 获取数据
      键盘文件 → 读 → 从键盘文件中读取数据
```

> **显示器和键盘也是"文件"**：往"显示器文件"写 = 打印；从"键盘文件"读 = 输入。
> 验证：终端本身就是文件 `/dev/pts/`（下一节权限课的板书里也会看到 `crw--w---- ... /dev/pts/0` 这些）。

### 3.6 `echo` / `printf`

```bash
[root@bite-alicloud lesson4]# echo "hello" "你好"
hello 你好

[root@bite-alicloud lesson4]# printf "hello world, %d, %f\n" 3 3.14
hello world, 3, 3.140000
```

- `echo`：原样输出，默认在末尾自动加换行
- `printf`：**格式化输出**（和 C 语言的 `printf` 一个东西），要自己写 `\n`

### 3.7 输出重定向 `>` 与 `>>`

```bash
[root@bite-alicloud lesson4]# echo "hello 你好" > hello.txt
[root@bite-alicloud lesson4]# ls -l
-rw-r--r-- 1 root root 13 Jul 14 16:09 hello.txt
[root@bite-alicloud lesson4]# cat hello.txt
hello 你好
```

板书标注的要点：

- **写入的时候，都是先清空再写入！！**
- **把 echo 输出的内容，写到了 hello.txt**
- **本来是要求写到显示器文件 → 输出重定向**

**用 `>` 可以清空文件**：

```bash
[root@bite-alicloud lesson4]# >hello.txt
[root@bite-alicloud lesson4]# ls -l
-rw-r--r-- 1 root root 0 Jul 14 16:18 hello.txt     ← 清空文件的！！！
```

**追加重定向 `>>`**：

```bash
[root@bite-alicloud lesson4]# echo "aaaaaaaaaaaaabbb" >> hello.txt
[root@bite-alicloud lesson4]# cat hello.txt
aaaaaaaaaaaaabbb
aaaaaaaaaaaaabbb        ← 上一次的内容还在，新的追加在后面
```

| 符号 | 作用 | 是否清空原内容 |
| --- | --- | --- |
| `>` | 输出重定向 | **清空后写入** |
| `>>` | 追加重定向 | 保留，追加到末尾 |
| `<` | 输入重定向 | — |

### 3.8 输入重定向 `<`

```bash
[root@bite-alicloud lesson4]# cat < hello.txt
aaaaaaaaaaaaabbb        ← 输入重定向！！
```

> 本来 `cat` 是"从键盘读、往显示器写"；`< hello.txt` 把"读"的来源换成了文件。

### 3.9 管道 `|`

```bash
[root@bite-alicloud lesson4]# head -5011 hello.txt | tail -10
hello 2001
hello 2002
hello 2003
...
```

板书对管道的解释：

```text
管道：入口 ── 出口
       传送资源的！
       数据是资源
       内存级
```

> **管道把"前一个命令的输出"直接当作"后一个命令的输入"**，不需要经过磁盘文件。
> 结合前面的"一切皆文件"，板书追问：**管道是文件吗？？**（后面 `管道文件 p` 那一课会回答）

### 3.10 `date`：时间与格式化

```bash
[root@bite-alicloud lesson4]# date +%Y-%m-%d_%H:%M:%S
2025-07-14_18:00:03
```

格式化占位符（板书原表）：

| 占位符 | 含义 |
| --- | --- |
| `%H` | 小时（00..23） |
| `%M` | 分钟（00..59） |
| `%S` | 秒（00..59） |
| `%X` | 相当于 `%H:%M:%S` |
| `%F` | 日期（年-月-日） |
| `%m` | 月份（01..12） |
| `%Y` | 完整年份（0000..9999） |
| `%s` | 时间戳（unix timestamp） |

**时间戳：从 1970-01-01 00:00:00 UTC 起算的秒数**，每秒 +1：

```bash
[root@bite-alicloud lesson4]# date +%s
1752487293
[root@bite-alicloud lesson4]# date +%s
1752487295
[root@bite-alicloud lesson4]# date +%s
1752487296
...
[root@bite-alicloud lesson4]# date +%s
1752487299
```

> 板书写了 `date +%s` 的双向关系：`时间戳 ⇄ 日期时间`（后面讲日志、缓存过期、`stat` 的时间都是这个数）。

---

## 四、外壳理解：shell 到底是什么

### 4.1 广义与狭义的操作系统

```text
广义操作系统 = 操作系统内核 + 外壳程序（shell 命令行、图形化界面）+ 必要的应用软件

狭义操作系统 = 操作系统内核！
```

> 所以"Windows 是一个操作系统"其实是广义说法；**严格意义上操作系统只有那个内核**。这也解释了 1.3 里那句"所有的操作系统都是白屏黑字命令行"——图形界面和命令行一样，都是外壳。

### 4.2 用户 → shell → 内核

```text
        用户
         ↓
  外壳程序（shell 命令行 / 图形化界面）
  ↓ 命令行解释器
  ↓ 系统调用接口
   Linux 内核
         ↓
       硬件
```

板书的三个关键结论：

1. **人不擅长和内核直接交互！**
2. **外壳可以变相地保护内核！！！**
3. **外壳程序 = 命令行解释器**

> "保护内核"的意思是：内核只暴露少量、严格、安全的系统调用接口；外层的 shell 负责把用户输入的字符串翻译成这些调用，**用户永远不能直接乱搞内核**。

### 4.3 bash 就是一个外壳程序

```bash
[root@bite-alicloud ~]# ls /usr/bin/bash
/usr/bin/bash                    ← 外壳程序 bash
```

```bash
[root@bite-alicloud lesson5]# ls /usr/bin/ls -l
-rwxr-xr-x 1 root root 117608 Nov 17 2020 /usr/bin/ls
```

> 顺势得出第二章那个问题的答案：**我们未来所学的大部分指令，本质都是可执行文件！！！**（`ls` 就是 `/usr/bin/ls` 这个可执行文件）

### 4.4 王婆、老李、小李与如花：一个讲进程的故事

板书用一段"相亲"比喻讲清楚了 shell、内核与父子进程：

```text
［操作系统内核］ = 一堵墙（红色粗线），把"外面的人"和"内核"隔开

王婆（媒婆）  ← 外壳程序
老李、小李    ← 用户输入的命令
如花          ← 真正要执行的东西

用户输入命令 → 王婆 → （王婆去问）如花是谁？→ 小李是谁呀？command not found!
                          ↓
                    子进程失败
                          ↓
                    父进程没退出
                          ↓
                  王婆有底线，保护了如花！
```

几个点：

- **如果 shell 外壳是媒婆，那么 bash 就是王婆**（`/usr/bin/bash` 就是那个"王婆"）
- 外壳是用户和内核之间的"中间人"，**它替你去问、去找；找不到就报 `command not found!`**
- 讲解里还出现了 `子进程失败 / 父进程没退出`——这是后面"进程控制 / 进程等待"的伏笔

### 4.5 `find` 与 `which`：系统是怎么"找到"命令的

```bash
[root@bite-alicloud lesson5]# find / -name XXX     # 会去目录树中进行查找

which 指令        # 系统级的命令 → 找出该命令在哪个路径下
find /usr/bin -name XXX
ls /usr/bin/XXX
```

> 板书明确写了 **"不建议这么做！"**——`find / -name` **会污染系统的命令池**（遍历整棵目录树，非常慢，还可能在 `/proc`、`/sys` 上乱跑）。找命令用 `which` / `whereis` 就好。

**执行一个命令的完整查找顺序**：

```text
执行命令 → 优先查找"别名" → 不存在再到 /usr/bin/XX 里找
           ↓
       别名是内存级的，重新登录就没有了！
```

> 这就是 `alias` 的本质：**只是当前 shell 进程内存里的一个字符串替换**，不落盘、不跨会话。
> 板书的追问：`grep 的应用场景？`（用别名简化长命令，比如 `alias ll='ls -l'`）

### 4.6 打包、压缩与传输

板书先问三个问题：

```text
打包？压缩？为什么？
        ↓
   网络传输（文件变小传得快）
        ↓
   下载效率
        ↓
   解压在解包
```

**zip / unzip**：

```bash
# zip -r lesson5.zip lesson5
# unzip lesson5.zip
```

```bash
# zip testl.zip test2/
# unzip testl.zip -d . /tmp        ← -d 指定解压目录
```

**tar（Linux 上最常用）**：

```bash
tar czf target.tgz XXX        # c=创建 z=gzip 压缩 f=指定文件名
tar xzf target.tgz            # x=解压
tar xzf lesson5.tgz -C test   # -C 指定解压到哪个目录
```

> 参数可以用"绝对 or 相对路径"，但**打包时建议用相对路径**（否则解压出来是一整条绝对路径）。记忆法：**c 创建、x 解压、t 查看；z 走 gzip；f 后面一定接文件名。**

**跨机器传输 `scp`**：

```bash
scp lesson5.tgz whb@8.137.19.140:/home/whb/code/116_117
```

```text
scp 源文件 用户名@目标云服务器:目标路径
```

> 板书描述：**passwd 输入的，是目标云服务器那个账号的密码（密码不回显）**。

### 4.7 `uname -r`：看内核版本与 CPU 架构

```bash
[root@bite-alicloud 116_117]# uname -r
3.10.0-1160.119.1.el7.x86_64
```

```text
x86     → 32 位
x86_64  → X64（64 位）
```

> `el7` = Enterprise Linux 7（CentOS 7）；`x86_64` 就是架构。**装软件时"架构必须对上"**（后面 `yum install` 会用到）。

### 4.8 三个必须会用的热键

| 热键 | 作用 |
| --- | --- |
| `Tab` | 命令/路径自动补全（连按两次列出候选） |
| `Ctrl + C` | 终止当前正在执行的操作（进程收到信号） |
| `Ctrl + R` | 搜索历史命令（输入关键字就能翻出以前打过的长命令） |

---

## 五、文件权限

### 5.1 三个基本问题

1. **什么是权限**：访问资源，能还是不能的问题！
2. **为什么要有权限**：Linux 是多用户操作系统，同时会有多个人登录访问（普通用户与 root）。**权限本质是为了更好地进行用户管理。**
3. **权限 = 人 + 文件属性**
   - 权限是针对特定群体的 → 与人有关（普通用户 or root）
   - 目标主体**必须天然具备对应的属性，才能访问**
   - `Linux — 一切皆文件` → 文件的三属性：**可读(r)、可写(w)、可执行(x)**

### 5.2 用户：root 与普通用户

```bash
]# adduser
]# passwd
```

| 用户类别 | 说明 |
| --- | --- |
| 超级用户 `root` | 不受权限约束，属于 Linux 系统中的特权级别 |
| 普通用户 | 受权限约束 |

**`su` 切换用户**：

```bash
[root@bite-alicloud ~]# su whb
```

```text
超级用户 root ──(可以切换)──> 普通账号 ──┬─> 普通1
                                    └─> 普通2

su 账号名                ──> 不需要密码（root 切普通）
su root / su / su -      ──> 需要 root 密码 ──> 管理员
```

> `su` 是"切换用户，让目标用户以重新登录的方式切换"。`su -` 会**连环境变量一起切换**（更彻底）。

### 5.3 三种角色：权限是依附于角色的

> **权限是依附于角色的！**

| 角色 | 说明 |
| --- | --- |
| **拥有者角色**（owner） | 角色是需要人来扮演的！（谁创建/被指定为属主） |
| **所属组角色**（group） | 对文件权限进行**局部范围的、组级别的管理** |
| **other 角色** | 剩下所有人 |

```bash
[root@bite-alicloud lesson6]# ll
total 0
-rw-r--r-- 1 root root 0 Jul 19 16:13 test.txt
```

`root root` 两处（板书红圈）：**左为拥有者，右为所属组**。

**为什么要有所属组？**

> 假设脱离权限，只有拥有者 owner 和 other，没有 group：
> 组（如 `www` 组里有"张三""李四"）要共享文件，就只能把权限放给 **other**——**一旦给了 other，全机器所有人都能访问了**。
> 有了 group，就可以"**只给这一组人**"——这是"局部范围的组级别管理"。

### 5.4 三个细节

**细节 1：权限只会验证一次！**

```bash
[whb@bite-alicloud 116_117]$ ll
total 4
-rw-r--r-- 1 whb whb 38 Jul 19 17:00 test.txt
[whb@bite-alicloud 116_117]$ cat test.txt
cat: test.txt: Permission denied
[whb@bite-alicloud 116_117]$ echo "aaa" > test.txt
bash: test.txt: Permission denied
```

> **用户访问文件的时候，确定自己相对于文件的身份角色，只会验证一次！**
> 也就是说：先判定你属于哪个角色（owner / group / other），然后**只用那一组权限**判一次。哪怕你在别的组里有权限也不叠加。

**细节 2：关于"可执行"**

文件要被执行，需要两件事：

1. 该文件本身就是一个可执行的文件（ELF 等）
2. 文件本身具有**可执行权限**

> **给 X 权限，不是能让你执行，而是当你想执行的时候，OS 允许你执行。**

**细节 3：文件的拥有者，具有修改文件权限的能力**（→ 所以 `chmod` 由 owner 做）

### 5.5 权限串怎么读

```bash
[whb@bite-alicloud 116_117]$ ll
total 0
-rw-r--r-- 1 whb  whb  0 Jul 19 15:27 hello.txt
-rw-r--r-- 1 root root 0 Jul 19 16:21 whb.txt
-rw-r--r-- 1 whb  whb  0 Jul 19 15:28 world.txt
-rw-r--r-- 1 root root 0 Jul 19 15:28 world.txt
```

权限串 `-rwxrw-r--` 按位拆解（板书标了序号 1~7 与箭头）：

| 位置 | 位 | 归属 |
| --- | --- | --- |
| 第 1 位 | `-` | **文件类型**（`-` 普通文件） |
| 第 2~4 位 | `rwx` | **拥有者权限**（可读？可写？可执行？） |
| 第 5~7 位 | `rw-` | **所属组权限** |
| 第 8~10 位 | `r--` | **other 权限** |

**文件类型一览**（板书竖排列表，末注"见一见"）：

| 字符 | 类型 |
| --- | --- |
| `-` | 普通文件（源码、文本、动态库、可执行文件） |
| `d` | 目录 |
| `l` | 软链接（短链接） |
| `p` | 管道文件 |
| `b` | 块设备文件 |
| `c` | 字符设备文件 |

**rwx 对"文件"和"目录"的含义完全不同**（这是最容易被忽略的一点）：

| 权限 | 对普通文件（`-`） | 对目录（`d`） |
| --- | --- | --- |
| `r` | 是否可**读取内容** | 是否能**查看目录里有哪些文件名/属性**（`ls`） |
| `w` | 是否可**修改内容** | 是否能在目录里**新建、删除、重命名**文件 |
| `x` | 是否可**执行** | 是否能**进入该目录**（`cd`）、访问其中的文件 |

> 关键：**目录的 `x` 是"通行证"**。没有 `x`，即使有 `r` 也 `cd` 不进去。

### 5.6 `sudo` 与提权

普通用户确实会遇到"需要更高权限"的事：

```text
普通用户需要提高权限的事情
   ↳ 安装软件 → 拷贝到指定的目录下 → OS 不让普通账号来拷贝
        ↳ 提权
   ↳ 安装软件只能由管理员来做？？？不！！！
```

```bash
[whb@bite-alicloud 116_117]$ touch hello.txt
[whb@bite-alicloud 116_117]$ ls -l
-rw-r--r-- 1 whb whb 0 Jul 19 15:25 hello.txt

[whb@bite-alicloud 116_117]$ sudo touch hello.txt
[sudo] password for whb:
[whb@bite-alicloud 116_117]$ ls -l
-rw-r--r-- 1 whb whb 0 Jul 19 15:27 hello.txt

[whb@bite-alicloud 116_117]$ sudo touch world.txt
[whb@bite-alicloud 116_117]$ ls -l
-rw-r--r-- 1 whb  whb  0 Jul 19 15:27 hello.txt
-rw-r--r-- 1 root root 0 Jul 19 15:28 world.txt      ← 属主变成了 root！
```

> **`sudo` = 普通账号执行"提权级"的命令**。注意最后一行：`sudo touch world.txt` 创建出来的文件**属主是 root**（因为真的以 root 身份做了这件事）。

**普通用户默认用不了 `sudo`**：

```text
whb is not in the sudoers file. This incident will be reported.
```

板书围绕它的追问与结论：

- 你的系统让你用 sudo 吗？
- 我输入的是自己的密码？？（`[sudo] password for whb:` 要的就是**自己的**密码，不是 root 的）
- **普通账户默认是执行不了 sudo 的！！！**（云服务器/虚拟机默认配置不同）

```bash
[whb@bite-alicloud 116_117]$ ls /etc/sudoers
```

> **`/etc/sudoers` 就是"白名单"**：里面列了谁能用 sudo、能用哪些命令。要让某个用户能用 sudo，就得把这个用户加进去（实践上用 `visudo` 编辑）。

### 5.7 `chmod` / `chown` / `chgrp` 与"没权限"的表现

```bash
[root@bite-alicloud 116_117]# ll
total 4
-rwxrw-r-- 1 whb whb 50 Jul 19 17:23 test.txt

[root@bite-alicloud 116_117]# chown whb:whb test.txt      # 改属主:属组
[root@bite-alicloud 116_117]# chgrp root test.txt         # 只改属组
```

普通用户去改属主/属组 —— **一律失败**：

```bash
[whb@bite-alicloud 116_117]$ chown zpw test.txt
chown: changing ownership of 'test.txt': Operation not permitted
[whb@bite-alicloud 116_117]$ chgrp zpw test.txt
chgrp: changing group of 'test.txt': Operation not permitted
[whb@bite-alicloud 116_117]$ chown root test.txt
chown: changing ownership of 'test.txt': Operation not permitted
[whb@bite-alicloud 116_117]$ chgrp root test.txt
chgrp: changing group of 'test.txt': Operation not permitted
```

> **为什么？文件给另一个人或者组，需要别人允许！**
> → **需要超级用户的权限了！**（细节 4）
>
> 注意：**哪怕文件本来就是你自己的**，也不能随便"送给别人"——因为这会绕过对方的意愿和磁盘配额。

### 5.8 顺带认识块设备与字符设备

```bash
[whb@bite-alicloud 116_117]$ ls /dev/vda -l
brw-rw---- 1 root disk 253, 0 Mar 12 18:42 /dev/vda
brw-rw---- 1 root disk 253, 1 Mar 12 18:42 /dev/vda1
```

- `b` → **块设备文件，硬盘**
- 箭头标注：**支持随机读写，块**

```bash
[whb@bite-alicloud 116_117]$ ls /dev/pts/ -l
total 0
crw--w---- 1 root tty 136, 0 Jul 19 15:16 0
crw--w---- 1 root tty 136, 1 Jul 19 15:51 1
crw--w---- 1 root tty 136, 2 Jul 19 17:52 2
crw--w---- 1 root tty 136, 3 Jul 19 17:47 3
crw--w---- 1 whb  tty 136, 4 Jul 19 17:52 4
```

- `c` → **字符设备**（显示器、键盘）
- 箭头标注：**按照顺序 0、不支持随机读写**

> 两者最右的 `253, 0` / `136, 0` 是**主设备号,次设备号**。`/dev/pts/0~4` 就对应"5 个终端"——即前面说的"终端本身也是文件"。

### 5.9 三个子问题（留给下一次课）

1. 进入一个目录需要什么权限？？
2. 为什么我们新建一个文件，默认权限是我们所看到的那个样子？
3. 没有 rwx，我能删除这个文件吗？

---

## 六、目录权限、umask、粘滞位与包管理

### 6.1 问题 1：进入一个目录需要什么权限

> 问题 1：目录权限问题：想要进入一个目录需要什么权限

- ✓ **r**：用户是否有权利查看到目录下的**文件属性**
- ✓ **w**：决定了特定用户是否有权利在该目录下**新增、删除文件，和修改文件名**
- ✓ **x**：进入目录（`cd`）的前提

```bash
[whb@bite-alicloud 116_117]$ ll
total 8
drwxrwxr-x 2 whb whb 4096 Jul 22 14:44 lesson7
[whb@bite-alicloud 116_117]$ cd lesson7
-bash: cd: lesson7: Permission denied
```

### 6.2 问题 2：为什么新建的文件/目录权限是这个样子

> 对于普通文件，起始权限是 **666** 开始
> 对于目录文件，起始权限是 **777** 开始

但实际看到的：

```bash
[whb@bite-alicloud lesson7]$ ll
total 4
drwxrwxr-x 2 whb whb 4096 Jul 22 15:03 dir        ← 775，不是 777
-rw-rw-r-- 1 whb whb    0 Jul 22 15:03 ha.txt     ← 664，不是 666
-rw-rw-r-- 1 whb whb    0 Jul 22 15:03 test.c
```

**原因：权限掩码 `umask`**（为了对权限进行细粒度控制）：

```bash
[whb@bite-alicloud lesson7]$ umask
0002
```

板书的竖式演算（二进制/八进制对照）：

```text
        普通文件 666                        目录 777
      110 110 110                       111 111 111
   &  111 111 101   ← ~umask           & 111 111 101   ← ~umask
  ─────────────────                  ─────────────────
      110 110 110 -> 664               111 111 101 -> 775
        6   6   4                         7   7   5

   000 000 010 -> umask        000 000 010 -> ~umask
```

**两条结论（务必记牢）**：

> - 凡是出现在权限掩码中的权限，**最终都应该在起始权限中去掉**；
> - **最终权限 = 起始权限 & ( ~umask )**
> - **误区：这里的"去掉"不能理解为减法**（是**按位与反**，不是减）。

### 6.3 问题 3：粘滞位（`t` 位）

先看一个矛盾：

```text
指定目录下，一个其他人新建的文件（即使它是 root 的），不想让删？
但是可能我们普通用户可以删除！！！
→ 取决于谁的权限？（答案是"目录的 w 权限"，不是文件自己的权限）
```

**共享文件不能放在谁的家目录下**：

```bash
[whb@bite-alicloud home]$ ls /home/ -dl
drwx------ 16 whb whb 4096 Jul 19 15:51 /home/whb      ← 700！别人根本进不来
drwxr-xr-x  7 zjy zjy 4096 Oct 14  2024 /home/zjy
drwxr-xr-x  7 zjy zjy 4096 Oct 14  2024 /home/qjh
```

> - **一定不会在任意用户的家目录下！！**（家目录是 `drwx------`，私有）
> - **→ 系统级别的路径下**（比如 `/tmp`）

**但共享目录又带来新问题**：

> 在一个共享目录下，whb 新建文件和其他人共享，别人可以删！
> 既要可以共享，又不想让除了 owner 之外的其他人都删 → **粘滞位！！！**

```bash
drwxrwxrwt 2 root root 4096 Jul 22 16:01 tmp       ← 末位是 t 而不是 x
```

> 给 other 设置的一种权限管理级别：
> **指定目录下，每个人新建文件，删除的时候只能删除自己的，不能因为 dir 的 w 权限而删除别人的文件！**
> **通常用来做文件共享的场景！**

（`/tmp` 就是系统自带的、带粘滞位的共享目录。）

### 6.4 Linux 下安装软件的三种方式

| 方式 | 说明 |
| --- | --- |
| 1. 源代码安装 | → tarball（`./configure && make && make install`） |
| 2. `rpm` 安装 | 直接装二进制包，但**不管依赖** |
| 3. **包管理器安装** | **最佳实践** |

### 6.5 yum 与 apt

```text
centos  → yum
apt     → apt        （板书原样：Debian/Ubuntu 系）
```

> - `yum`（Yellow dog Updater, Modified）是 Linux 下非常常用的一种包管理器，主要应用在 **Fedora、RedHat、CentOS** 等发行版上。
> - `apt`（Advanced Package Tool）应用在 **Debian、Ubuntu** 等发行版上。

| 发行版阵营 | 包管理器 | 全称 | 适用发行版 |
| --- | --- | --- | --- |
| RedHat 系 | `yum` | Yellow dog Updater, Modified | Fedora、RedHat、CentOS |
| Debian 系 | `apt` | Advanced Package Tool | Debian、Ubuntu |

> 板书红叉与追问：**"安装软件，还需要了解 Linux 发行版吗？"**
> 答案：**需要**——因为包管理器和源的格式不同；但如果你只用一种发行版（课上固定 CentOS 7 / Ubuntu 20.04），只需要记一套命令。

### 6.6 如何评估一款操作系统的好坏

> 操作系统被设计出来之后，最重要的事情是什么？
> **必须有更多的人使用！！！** ← 最原始的动力

```text
            操作系统生态
   ┌───────────────────────────────┐
   │  社区论坛   社区文档           │
   │  软件体系                      │
   │  算法、数据结构                │
   │  操作系统自身                  │
   │  基于…的客户端/用户            │
   └───────────────────────────────┘
              ↓ 形成使用
   手机 → 目的 → 进一步 → 社区/文档/论坛/资料
              ↓
   有人愿意在特定的 OS 上，编写特定的软件，适应不同的群体！
```

> 这和第一章"生态"那张图是同一个道理：**用的人多 → 有人愿意为它写软件 → 更好用 → 用的人更多**。

### 6.7 包管理器是怎么工作的

**第一层：你的机器 ←→ 软件包服务器**

```text
┌──────────────┐                  ┌──────────────────┐
│  你的云服务器  │ ───────────→    │  Test 软件包+     │
└──────────────┘                  │ 软件包1 … 软件包b │
                                  └──────────────────┘
                                        软件包服务器
```

**第二层：包管理器 → 找到下载地址 → 镜像站**

```text
┌────────────┐
│  包管理器   │  yum / apt
└────────────┘
   解决依赖、下载、安装/卸载
        │
   我应该去哪里？？→ 找一下下载地址
        ↓
   软件包仓库（repo）
        ↑
   aliyun镜像站 / CentOS镜像站 / 清华源 / 华为云
        （红色批注：镜像、url）
```

**第三层：源配置是"内置"的**

> 板书明确写：**"你们不用做"**
> - 包管理器**自带软件源配置**（`软件内置配置文件 → 软件源配置`）
> - 需要换源时改的就是这个配置文件里的 **URL**（内网、镜像站）

CentOS / Ubuntu 的源配置文件位置（本章要点，实践中第一件事就是换源）：

| 发行版 | 配置文件 |
| --- | --- |
| CentOS | `/etc/yum.repos.d/CentOS-Base.repo` |
| Ubuntu | `/etc/apt/sources.list` |

**依赖列表：为什么包管理器"什么都能装"**

```text
                      (云) libc.so
                    ↗ 依赖
┌──────────────┐  依赖        (云) evince.xx
│  你的软件 /   │ ────→
│  你要的软件   │  依赖  ┌───────────────┐  依赖  ┌───────────┐
└──────────────┘ ────→ │ libstdc++.so  │ ────→ │ libc.so.6 │
                       └───────────────┘        └───────────┘
                              │ 依赖
                              ↓
                        (云) xxx.so

        ══════════════════════════════════
        │          依 赖 列 表              │
        ══════════════════════════════════
```

要点：

- 你要装的软件会依赖一堆 `.so` 库；
- 这些库**自己还依赖别的库**（`libstdc++.so → libc.so.6`），形成**依赖链**；
- 包管理器的核心能力就是：**读依赖列表 → 一次性把所有依赖都下下来 → 按顺序安装**。
  （这正是"包管理器是最佳实践"的原因——`rpm` 手动装会被依赖折磨死。）

---

## 七、命令速查表

### 7.1 文件与目录

| 命令 | 作用 | 常用参数 |
| --- | --- | --- |
| `pwd` | 显示当前路径 | — |
| `whoami` | 显示当前用户 | — |
| `ls` | 列目录 | `-l` 详情、`-a` 含隐藏、`-d` 看目录本身 |
| `cd` | 切换目录 | `cd ..` 上级、`cd ~` 家目录、`cd -` 上次 |
| `mkdir` | 建目录 | `-p` 建多级 |
| `touch` | 建空文件 | — |
| `rm` | 删除 | `-r` 递归、`-f` 强制 |
| `cp` | 拷贝 | `-r` 目录 |
| `mv` | 移动/重命名 | — |
| `tree` | 树状显示 | `yum/apt install -y tree` |
| `stat` | 文件属性全貌 | — |
| `file` | 真实文件类型 | — |
| `find` | 按名字找 | `find 路径 -name xxx` |
| `which` | 找命令在哪 | — |

### 7.2 查看与文本

| 命令 | 作用 |
| --- | --- |
| `cat` | 打印全部内容（小文件） |
| `less` / `more` | 分页查看（大文件、日志，推荐 `less`） |
| `head -n` / `tail -n` | 前/后 n 行 |
| `echo` / `printf` | 输出 / 格式化输出 |
| `grep` | 按内容过滤 |
| `man` | 手册（`man 1` 命令、`man 2` 系统调用、`man 3` 库函数） |
| `date +格式` | 时间；`%s` 时间戳 |

### 7.3 重定向与管道

| 符号 | 作用 |
| --- | --- |
| `>` | 输出重定向（**先清空再写**） |
| `>>` | 追加重定向（保留原内容） |
| `<` | 输入重定向 |
| `\|` | 管道（前者输出 → 后者输入，内存级） |

### 7.4 压缩与传输

| 命令 | 作用 |
| --- | --- |
| `zip -r x.zip dir` / `unzip x.zip -d 目录` | zip 打包/解包 |
| `tar czf x.tgz dir` | 打包 + gzip 压缩 |
| `tar xzf x.tgz [-C 目录]` | 解压 |
| `scp 文件 用户@IP:目标路径` | 跨机器传输 |
| `uname -r` | 内核版本 + 架构（`x86_64` = 64 位） |

### 7.5 权限

| 命令 | 作用 |
| --- | --- |
| `chmod` | 改权限（`chmod 755 f` / `chmod u+x f`） |
| `chown 用户:组 文件` | 改属主与属组（**要 root**） |
| `chgrp 组 文件` | 改属组（**要 root**） |
| `umask` | 看/设权限掩码（最终权限 = 起始 & ~umask） |
| `su [-] 用户` | 切换用户 |
| `sudo 命令` | 以 root 权限执行单条命令（要 /etc/sudoers 允许） |

### 7.6 一张表记住"三板斧"

| 我想… | 怎么做 |
| --- | --- |
| 知道自己在哪 | `pwd` |
| 知道我是谁 | `whoami` |
| 知道有什么 | `ls -la` |
| 知道它是什么 | `file` + `stat` |
| 找到它 | `which`（命令）/ `find`（文件） |
| 看清大文件 | `less` |
| 取一段 | `head`/`tail` + `grep` |
| 改权限 | `chmod`（要 root 改属主用 `chown`） |
| 装软件 | `yum install -y xxx` / `apt install -y xxx` |

### 7.7 最容易踩的坑

1. **`rm -rf` 不可逆**：删之前先 `pwd` 看清楚在哪。
2. **`>` 会清空文件**：想追加一定用 `>>`。
3. **执行当前目录的程序要写 `./`**：`PATH` 里没有当前目录。
4. **后缀不代表类型**：用 `file` 看，别靠 `.txt` / `.exe`。
5. **目录的 `x` 是通行证**：没有 `x`，有 `r` 也进不去。
6. **`umask` 是"按位与反"，不是减法**。
7. **`chown` 给普通用户 → `Operation not permitted`**：需要 root。
8. **`sudo` 要输自己的密码**，不是 root 的；且要在 `/etc/sudoers` 白名单里。
9. **`find / -name` 别乱用**：遍历整棵树，又慢又吵。
10. **相对路径会随 `cd` 失效**：配置文件里一律写绝对路径。

---

> 后续课程（超出本篇 5/18~7/22 范围）：`20250723_vim`（编辑器）、`20250727_gcc`（编译）、`20250730_make`、`20250731_进度条`、`20250801_gdb`（调试），之后进入进程与网络。
