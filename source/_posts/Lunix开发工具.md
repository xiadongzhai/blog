---
title: Lunix开发工具
date: 2026-11-27 18:14:20
tags: [Linux, vim, gcc, makefile, gdb, git, 动态库]
description: 'Linux 开发工具链笔记（2025-07-23 ~ 08-01）：vim 三种模式与命令、gcc/g++ 四阶段编译、动态链接与静态链接、make/makefile 依赖关系与 mtime 判定、回车换行与缓冲区的进度条原理、git 版本控制与工作区、gdb 调试命令表'
---

> 内容来源：`D:\computer source\Lunix\116_117-master\板书`（2025 年 7 月 23 日 ~ 8 月 1 日，共 5 次课）
>
> - `2025-07-23_vim`：包管理命令速览与换源、**IDE 与工具链的拆解**、vim 的三种模式与模式转换图、命令模式（移动/复制/粘贴/删除/替换/撤销）、批量注释与批量去注释、底行模式（保存退出/搜索/`!cmd`/分屏）、`.swp` 临时文件、光标恢复
> - `2025-07-27_gcc`：`gcc` vs `g++`、**编译四阶段**（预处理 → 编译 → 汇编 → 链接）、`#pragma once` 与条件编译、头文件与库文件的存放位置、**动态库与静态库对比**、`ldd`/`file`/`static` 链接实测
> - `20250730_make`：为什么需要 make、**依赖关系 + 依赖方法**、makefile 基本语法与隐式推导、`.PHONY`、`make` 为什么第二次不重新编译（mtime 判定）、`echo $?` 退出码
> - `20250731_进度条`：回车 `\r` 与换行 `\n`、**行刷新与缓冲区**、`fflush(stdout)`、倒计时与进度条、`usleep`、git 的历史与"只记变化"的核心思想
> - `20250801_gdb`：git 命令行的本地操作（`add`/`commit`/`log`/`push`/`pull`/`clone`/`status`/`.gitignore`）、工作区-暂存区-本地仓库、**为什么默认编译出来的程序不能调试**、`-g` 与 ELF 的 `.debug_*` 段、`readelf`、**gdb 命令表**、调试的本质
>
> 说明：板书中没有 7 月 21 日的文件，本篇按"开发工具"这条主线收录 7/23（vim）→ 7/27（gcc）→ 7/30（make）→ 7/31（进度条 + git 概念）→ 8/1（git 命令 + gdb）五次课；7/22 的 `yum/apt` 已归入 `Lunix基本指令.md`。

## 目录

- **一、先看清全景：IDE 其实是一堆工具**
- **二、vim：只用来写代码的编辑器**
- **三、gcc / g++：四阶段编译**
- **四、库：动态链接与静态链接**
- **五、make / makefile：一键自动化构建**
- **六、进度条：回车换行与缓冲区**
- **七、git：版本控制器**
- **八、gdb：调试器**
- **九、速查表与易错清单**

---

## 一、先看清全景：IDE 其实是一堆工具

> **Linux 下开发工具集！—— 只用来写代码的！！！**

| 环节 | 工具 |
| --- | --- |
| 写代码 | 编辑器：`vim`、`vscode` |
| 编译代码 | `gcc` / `g++` |
| 调试 | `gdb`、`cgdb` |
| 构建工具 | `makefile`、`make`、`cmake` |
| 版本管理 | `git` |

> **所谓 IDE，只是把这些工具"打包 + 加了个图形界面"。** 在 Linux 上它们是一个个独立的命令，这也是为什么"会命令行"才是真的会 Linux。

### 1.1 顺手复习：包管理命令与换源

```bash
sudo yum install -y sl
sudo yum list | grep XXX
sudo yum remove -y XX

sudo apt list | grep sl
sudo apt install -y sl
sudo apt remove -y sl
```

**更新源（CentOS）**：

```bash
yum clean all
yum makecache
yum -y install epel-release
```

> 板书的灵魂问题：**"我怎么知道，我要装什么软件？？"**
> 答案：**需求驱动！** —— 先遇到问题（比如"想装个 tree 看目录结构"），再去搜/问（板书写的"deepseek 帮你想成安装指令"），而不是背软件清单。

包管理器的三层结构（复习）：

```text
你的云服务器                软件包服务器
┌──────────┐  拷贝、下载请求  ┌──────────────┐
│ 包管理器  │ ──────────────→ │ 软件包1       │
│ yum/apt  │                 │ 软件包2  ...  │
│ 配置文件: │ ←── 下载返回 ── │ (BASE soft)   │
│ 下载/安装/卸载            └──────────────┘
└──────────┘                        ↑
                            扩展包服务器 ┌─ 开发者提交源码 → 编译 → 上传
   镜像站（aliyun/清华/华为云）
```

---

## 二、vim：只用来写代码的编辑器

### 2.1 先确认 vim 在哪、配置在哪

```bash
[zhangsan@bite-alicloud ~]$ which vim
/usr/bin/vim
[zhangsan@bite-alicloud ~]$ pwd
/home/zhangsan

[zhangsan@bite-alicloud ~]$ ls -al
total 32
drwx------  2 zhangsan zhangsan 4096 Jul 27 14:43 .
drwxr-xr-x  5 root     root     4096 Jul 27 14:38 ..
-rw-------  1 zhangsan zhangsan 1818 Nov 15  2021 .bash_logout
-rw-r--r--  1 zhangsan zhangsan  193 Nov 15  2021 .bash_profile
-rw-r--r--  1 zhangsan zhangsan  231 Nov 15  2021 .bashrc
-rw-r--r--  1 zhangsan zhangsan   30 Jul 27 14:41 test.c
-rw-r--r--  1 zhangsan zhangsan 4931 Jul 27 14:42 .viminfo
-rw-r--r--  1 zhangsan zhangsan    7 Jul 27 14:42 .vimrc
```

> 板书的重点：**对 vim 做基本配置，本质是把配置项写进 `/home/XXX/.vimrc` 配置文件里。**
> 而且——**vim 配置只在自己当前账号下有效！**（因为是家目录下的点文件）
>
> 所以"别人的 vim 用着顺手，我这边没效果"的答案永远是：**配置写进你自己的 `~/.vimrc`。**

### 2.2 三种模式（其实是 5 种）

> **vim 得多模式 —— 3 → 5**（一开始是 3 种，实际有 5 种）

```text
                    shift + v = 选中
                 ┌────────────────────┐
                 │   VISUAL BLOCK     │
                 └────────────────────┘
                     ↑  v(逐字符) / V(逐行) / Ctrl+v(块)
                     │
   ┌──────────┐  Esc   ┌──────────┐  shift + ;   ┌──────────┐
   │ 插入模式  │ ←────→ │ 命令模式  │ ───────────→ │ 底行模式  │
   │ (INSERT) │  a / i │ (默认)    │ ←─────────── │ (LAST)   │
   └──────────┘        └──────────┘     Esc      └──────────┘
                          ↕ Esc
                     ┌──────────┐
                     │ 替换模式  │  shift + R
                     └──────────┘
```

关键点：

1. **打开 vim 后默认处在"命令模式"**——此时敲键盘是在"下命令"，不是在输入文字。
2. `i`（insert）/ `a`（append）→ 进入**插入模式**，这时才能打字。
3. `Esc` → 回到**命令模式**。
4. 命令模式下按 `:`（即 `shift + ;`）→ 进入**底行模式**。
5. **插入模式不能直接到插入模式**（板书标注"不推荐！！"，实际上必须经 `Esc`）。

> 板书的哲学一句：**"围绕命令展开"** —— vim 的一切都是命令，`Esc` 是你永远的安全出口。

### 2.3 命令模式：移动与编辑

**移动**

```text
n + shift + g  = nG    定位到第 n 行（shift+g = G：进入文本末尾）
gg                     进入文本开始
shift + 4 = $          定位到当前行结尾
shift + 6 = ^          定位到当前行的开始
h  j  k  l             左 下 上 右（等价方向键）
w                      按"单词"为单位向右移动
b                      按"单词"为单位向左移动
```

> 板书还专门问了一个问题：**"为什么 vim 在已经有了上下左右方向键的时候，还引入了 h j k l？"**
> 原因很简单：**手指不用离开主键区**。方向键在键盘右下角，来回切换会让盲打手位被打断；`hjkl` 就在右手食指旁，手不离键速度才快。（板书配了一张老式键盘的照片。）

**编辑**

```text
yy                     复制当前行
p                      粘贴到当前行的下一行
n + dd                 删除当前行（n 行）
shift + ～             大小写快速转换
r                      替换光标所在位置的一个字符（替换单个字母）
shift + r = R          进入替换模式（可批量替换内容）
x                      删除光标所在字符（向右删除）
shift + x = X          向左删除
u                      撤销操作
ctrl + r               对"撤销"进行撤销（有一次"后悔的后悔"）
```

> **重要提醒（板书红字）**：**一旦退出，就无法撤销了！只要你没有退出 vim，随时可以后悔。**
> 因为撤销栈存在内存里（配合 `.viminfo` 只记少量状态），进程一退就没了。**所以"改坏了"的第一反应是别退，先 `u`。**

### 2.4 批量注释 / 批量去注释（Ctrl + v 块选择）

**批量注释**：

```text
1. ctrl + v        进入 VISUAL BLOCK（列选择）模式
2. 选中要注释的区域（j/k 上下扩展）
3. shift + i       进入插入模式（在块首插入）
4. 输入 //
5. Esc             两秒后，选中的每一行行首都加上 //
```

**批量去注释**：

```text
1. ctrl + v        列选择
2. 选中行首的 // 那一列
3. d               删除
```

> 这个技巧的关键在第 3 步：`shift + i` 在**块模式下**是"对所有被选中行同时插入"，不是普通插入。

### 2.5 底行模式

```text
w                  保存
w!                 【强制】保存
q                  退出
q!                 【强制】退出（放弃修改）
wq!                【强制】保存并退出
/key + n           匹配搜索：/是向下搜，n 跳到下一个匹配
!cmd               不退出 vim，直接对代码进行编译和运行
vs 文件名          vs 分屏操作，ctrl + ww 切换分屏
set nu / set nonu  显示 / 关闭行号
```

**`!cmd` 的实战写法**（边写边编译，不用退出）：

```vim
:!gcc % && ./a.out
```

> `%` 在底行模式里就是"当前文件名"。这一条命令直接完成了"编译 + 运行"。

**多文件与分屏**：

```vim
:vs main.c          垂直分屏打开 main.c
Ctrl + ww           在两个分屏之间切换光标
```

### 2.6 两个必须知道的"事故"

**（1）`.swp` 临时文件**

> **如果 vim 打开文件时突然终端退出，vim 会形成临时文件，默认在当前路径下的一个 `.xx.swp` 临时文件。**

现象：下次再 `vim xx` 会弹出：

```text
E325: ATTENTION
Found a swap file by the name ".xx.swp"
```

处理方式：

| 选择 | 含义 |
| --- | --- |
| `[O]pen Read-Only` | 只读打开（先看内容） |
| `[E]dit anyway` | 不管它，直接编辑（有风险） |
| `[R]ecover` | 从 swap 恢复未保存的内容（**通常这是你要的**） |
| `[D]elete it` | 删掉临时文件（确认不需要恢复时） |
| `[Q]uit` | 退出 |

**（2）光标位置会被记住**

```vim
vim main.c +9        # 打开 main.c 并跳到第 9 行
```

> 板书记的现象：**"vim 退出时光标在第 n 行，那么再次打开，光标还在原来的位置。"**
> 这是 `.viminfo` 在起作用（记录上次的光标位置、搜索历史等）。

---

## 三、gcc / g++：四阶段编译

### 3.1 gcc 与 g++

> - **`gcc` 是 C 编译器**，只能用来编译 C 语言
> - **`g++` 是一个 C++ 语言编译器**
>
> **两者选项（参数）完全一样**，只是默认语言与链接的库不同。

### 3.2 编译四阶段

> **程序开始翻译，做完对原文件的预处理，就停下来！！**

```text
   code.c
     │  ① 预处理（Preprocess）   gcc -E code.c -o code.i
     ↓
   code.i
     │  ② 编译（Compile）        gcc -S code.i -o code.s
     ↓
   code.s
     │  ③ 汇编（Assemble）       gcc -c code.s -o code.o
     ↓
   code.o  ← 可重定向目标二进制文件
     │  ④ 链接（Link）           gcc code.o -o code
     ↓
   code    ← 可执行程序（镜像文件）
```

**一步到位（最常用）**：

```bash
gcc code.c -o code
```

**① 预处理**：`gcc -E code.c -o code.i`

四件事：

```text
a. 头文件展开
b. 去注释
c. 条件编译
d. 宏替换
```

实测：

```bash
[whb@bite-alicloud lesson8]$ gcc -E code.c -o code.i
[whb@bite-alicloud lesson8]$ ls -l
-rw-r--r-- 1 whb whb  504 Jul 27 15:19 code.i      ← 几百行的 .i
-rw-r--r-- 1 whb whb 16946 Jul 27 15:23 code.c
```

> **本质是对代码做裁剪！**
> 板书追问：**"为什么会有这个功能？"** → 讲到 `#pragma once` 与条件编译时给出了答案方向：**公司内部如何保护这种项目的源代码？**
>
> ```c
> #ifdef __CODE_H__
> #define __CODE_H__
> // ... 真正的实现
> #endif
>
> #pragma once
> ```
>
> 预处理阶段就把"不该给客户看的部分"裁掉了——这也是 C 语言"发布买库/买源码"这种商业模式的底层手段。

**② 编译**：`gcc -S code.i -o code.s`

> **程序开始翻译，从 C 语言翻译成为汇编语言，就停下来！**

**③ 汇编**：`gcc -c code.s -o code.o`

> 把汇编语言翻译成为**二进制文件（`.o`，可重定向的目标二进制文件）**。

**④ 链接**：`gcc code.o -o code`

> 站在巨人的肩膀上：把 `.o` 和**库**拼成最终的可执行文件（镜像文件）。

### 3.3 一个必然踩到的问题：`.o` 不能直接运行

```bash
[whb@bite-alicloud lesson8]$ ./code.o
bash: ./code.o: cannot execute binary file
```

> **为什么？** 因为 `.o` 里**只调用了库函数（如 `printf`）的声明，没有定义**——链接就是把"声明"接到"库里的实现"上。

```bash
[whb@bite-alicloud lesson8]$ gcc code.o -o code     # 链接后就能跑了
```

### 3.4 那些"编译器知道、你不知道"的位置

```bash
# 编译器在哪
[whb@bite-alicloud lesson8]$ which gcc
/usr/bin/gcc

# 头文件在哪
[whb@bite-alicloud lesson8]$ ls /usr/include/stdio.h
/usr/include/stdio.h

# 库文件在哪
[whb@bite-alicloud lesson8]$ ls /lib64/libc-2.17.so
/lib64/libc-2.17.so
```

```text
开发 C 语言的三个"家"：
1. 编辑器/编译器   /usr/bin/gcc、/usr/bin/g++
2. 头文件          /usr/include/stdio.h      ← 只提供【声明】
3. 库文件          /lib64/libc-2.17.so      ← 提供【实现】
```

> `#include <stdio.h>` 里的内容只是**声明**（告诉编译器 `printf` 长什么样）；
> **真正的实现**在 `libc-2.17.so` 里。这就是"库"的感性认识。

---

## 四、库：动态链接与静态链接

### 4.1 命名规则

| 平台 | 静态库 | 动态库 |
| --- | --- | --- |
| **Linux** | `.a` | `.so`（shared object） |
| **Windows** | `.lib` | `.dll` |

> 命名规则：`libname.a` / `libname.so` —— **`lib` 前缀 + 名字 + 后缀**。
> 例如 `libc.so.6` → 名字是 `c`（C 标准库），`.so.6` 是主版本号。

### 4.2 默认是动态链接

```bash
[whb@bite-alicloud lesson8]$ vim code.c
[whb@bite-alicloud lesson8]$ gcc code.c -o code
[whb@bite-alicloud lesson8]$ ll
-rwxr-xr-x 1 whb whb 8360 Jul 27 17:53 code
-rw-r--r-- 1 whb whb 6980 Jul 27 17:53 code.c

[whb@bite-alicloud lesson8]$ ldd code
    linux-vdso.so.1 =>  (0x00007fff239c3000)
    libc.so.6 => /lib64/libc.so.6 (0x00007f7d3ce61000)
    /lib64/ld-linux-x86-64.so.2 (0x00007f7d3cc73000)

[whb@bite-alicloud lesson8]$ file code
code: ELF 64-bit LSB executable, x86-64, version 1 (SYSV), dynamically linked (uses shared libs), for GNU/Linux 2.6.32, not stripped
```

> **gcc 默认编译，采用动态链接的方式。**
> 板书一句话：**"编译器（我们通常叫程序）动态链接是最佳实践！"** —— 因为省内存。

### 4.3 动态库到底是怎么"共享"的

板书的比喻（很形象）：

```text
你家 = 你的程序            我家 = 动态库
   ┌────────┐               ┌────────┐
   │ 程序    │ ── 链接 ──→   │ 动态库  │
   │ 张三    │               │ 张三    │
   └────────┘               └────────┘
   链接时只记住"门牌号/地址"，不复制内容

运行时 → 加载（把动态库映射进内存）→ 程序去动态库那块内存取数据
```

**关键结论**：

1. 动态链接时，可执行文件里**只保存动态库的"地址/符号引用"**，不复制代码。
2. 运行时由**加载器**把 `.so` 映射进进程内存，多个程序**共享同一份**。
3. **一旦库缺失，所有相关程序无法运行！** —— 这就是"依赖"的代价。

### 4.4 静态链接

```bash
# 先确认装了 C 静态库
[whb@bite-alicloud lesson8]$ ls /lib64/libc.a
/lib64/libc.a

[whb@bite-alicloud lesson8]$ gcc code.c -o code-static -static
[whb@bite-alicloud lesson8]$ ll
-rwxr-xr-x 1 whb whb   8360 Jul 27 17:53 code
-rw-r--r-- 1 whb whb   6980 Jul 27 17:53 code.c
-rwxr-xr-x 1 whb whb 861288 Jul 27 17:57 code-static      ← 体积暴涨 100 倍

[whb@bite-alicloud lesson8]$ ldd code-static
    not a dynamic executable

[whb@bite-alicloud lesson8]$ file code-static
code-static: ELF 64-bit LSB executable, x86-64, version 1 (GNU/Linux), statically linked, for GNU/Linux ...
```

> `-static` 是**强制要求 gcc 采用静态链接方式**。
> 如果系统没装静态库：

```bash
[whb@bite-alicloud lesson9]$ gcc code.c -o code-static
/usr/bin/ld: cannot find -lc
collect2: error: ld returned 1 exit status
```

> 报错 `cannot find -lc` 就是"找不到 C 静态库"（`-lc` 表示链接 `libc`）。

### 4.5 动态库 vs 静态库

| | 动态库（`.so`，动态链接） | 静态库（`.a`，静态链接） |
| --- | --- | --- |
| 链接方式 | 只记录地址，运行时加载 | 把用到的库代码**直接拷进**程序 |
| **优点** | **节省内存空间**（多进程共享一份） | **不需要库文件**，一旦编译成功，不依赖库 |
| **缺点** | 库缺失则程序崩溃，**依旧依赖动态库** | **可执行程序体积大，占用内存资源高** |
| 典型大小 | 8 KB | 861 KB（本例） |

> **"把库中的方法我们理解所调的（用到的），直接拷贝到我的程序中，叫静态链接！"**
> 所以静态链接的程序"扛饿"（拷到任何地方都能跑），但"胖"。

---

## 五、make / makefile：一键自动化构建

### 5.1 为什么需要 make

```bash
gcc code.c -o code       # 一个文件还行
```

但真实项目是几十个 `.c`：

```bash
gcc -c src44.c src55.c src78.c ...
gcc src44.o src55.o ... -o main
```

每次改一个文件都要手敲一遍？→ **make 一键自动化构建。**

> 板书的定义非常准：
> - **`make` 是一条命令**
> - **`makefile` 是一个文件**
> - **make 会根据 makefile 文件，自己的规则，完成自动化构建**

### 5.2 最小示例：见一见 make / makefile

**`Makefile`**

```makefile
test.exe: test.c
	gcc -o test.exe test.c
```

```bash
[whb@bite-alicloud lesson9]$ make
gcc -o test.exe test.c
```

> 注意：**`make` 会把它执行的那条命令回显出来**（这一点在排查问题时特别有用）。

### 5.3 依赖关系 + 依赖方法（核心概念）

```makefile
目标文件: 依赖文件列表
<Tab>依赖方法
```

```makefile
test.exe: test.c        # 依赖关系：test.exe 依赖 test.c
	gcc -o test.exe test.c   # 依赖方法：怎么从 test.c 得到 test.exe
```

板书用一段对话讲清了这对概念：

> **"我是你儿子"** → 这是**依赖关系**（你是我的父亲，我们之间有这种关系）
> **"打包"** → 这是**依赖方法**（具体怎么做才能拿到钱）

> **"这个世界，要完成任何一件事，都是依赖关系和依赖方法的关系！"**

**关键格式要求**：**依赖方法前面必须是 Tab，不是空格**（这是 Makefile 最经典的坑）。

### 5.4 从一条命令到完整工程

```makefile
test.exe: test.o                          # 形成目标（链接）
	gcc test.o -o test.exe

test.o: test.s                            # 汇编
	gcc -c test.s -o test.o

test.s: test.i                            # 编译
	gcc -S test.i -o test.s

test.i: test.c                            # 预处理
	gcc -E test.c -o test.i

.PHONY: clean                             # 伪目标
clean:
	rm -f test.exe test.i test.s test.o
```

```bash
[whb@bite-alicloud lesson9]$ make
[whb@bite-alicloud lesson9]$ make clean
```

> **"依赖方法即最简单的脚本！"** —— makefile 里每条依赖方法就是一条 shell 命令，可以写任何东西。
>
> **`make` 有隐式推导能力**：像 `test.o` 这种"由 `test.c` 编译得到"的规则，make 内置了默认推导（`gcc -c test.c -o test.o`），所以很多时候你可以只写链接那一条规则。

### 5.5 四个细节 + `.PHONY`

**细节 1：依赖关系必须存在，依赖文件列表可以为空**

```makefile
clean:                # 没有依赖文件，但冒号必须有
	rm -f test.exe
```

**细节 2：依赖方法可以是任何 shell 命令**（不只是 gcc）

**细节 3：`clean` 只是"利用 make 的自动化构建能力"去执行 `rm`**

> 在"构建工程"的视角里，**看起来就是清理项目**；**清理执行的本质，就是删除不需要的临时文件！**
> （顺带说：`make clean` 又叫"**目标名**"——`make` 后面跟谁，就执行哪个目标的规则。）

**细节 4：`make` 命令后面可以跟"目标名"**

```bash
make test.exe         # 只构建 test.exe
make clean            # 只执行 clean
```

### 5.6 为什么第二次 `make` 不干活了？

```bash
[whb@bite-alicloud lesson9]$ make
gcc -o test.exe test.c

[whb@bite-alicloud lesson9]$ make
make: 'test.exe' is up to date.        ← 什么都没干！
```

**原因：make 会比较"目标文件"和"依赖文件"的修改时间（mtime）**：

```text
源文件 test.c 的 mtime   vs   目标 test.exe 的 mtime

if (test.exe 比 test.c 新)  →  "已经是最新的" → 不重新编译
if (test.c  比 test.exe 新) →  源文件变了 → 重新编译
```

实测证明：

```bash
[whb@bite-alicloud lesson9]$ stat test.exe
  File: 'test.exe'
  Size: 8360        Blocks: 8          IO Block: 4096   regular file
  Device: fd01h/64769d  Inode: 1041888   Links: 1
  Access: (0777/-rwxrwxrwx)  Uid: ( 1000/    whb)   Gid: ( 1000/    whb)
  Access: 2025-07-30 10:17:01.927222121 +0800
  Modify: 2025-07-30 10:31:04.006175845 +0800      ← 目标更晚
  Change: 2025-07-30 10:31:04.006175845 +0800

[whb@bite-alicloud lesson9]$ stat test.c
  File: 'test.c'
  Size: 169         Blocks: 8          IO Block: 4096   regular file
  Modify: 2025-07-30 10:16:58.000000000 +0800      ← 源码更早
```

**`touch test.c` 之后**（touch 会把 mtime 刷新成当前时间）→ 再 `make`：

```bash
[whb@bite-alicloud lesson9]$ touch test.c
[whb@bite-alicloud lesson9]$ make
gcc -o test.exe test.c              ← 源文件变新了，重新编译！
```

> **结论：源文件是否需要重新编译？判断标准是"源文件和目标谁更新时间更晚"，比的是两个文件的 `Modify` 时间。**
> 这也解释了为什么 `make` 对"只是 `cat` 看了一眼"不敏感——**`access` 时间不算，只算 `modify`**。

### 5.7 `.PHONY`：伪目标，总是被执行

```makefile
.PHONY: clean
clean:
	rm -f test.exe test.i test.s test.o
```

> **`.PHONY` 用来修饰目标文件，声明它是一个"伪目标"。**
> **本质：总是被执行的！**

**为什么需要它？**

- `clean` 这个"目标"**永远不会真的生成一个叫 `clean` 的文件**，所以每次 `make clean` 都应该执行。
- 但如果目录里**恰好有个文件叫 `clean`**，且它比依赖文件新，make 就会认为"clean 已经是最新的"而不执行 → **加 `.PHONY` 就永远执行**。
- 更普遍的用处：**要执行"没有依赖文件"的目标，必须用 `.PHONY`**，否则 make 无法判断新旧。

> 板书的追问很有意思：**"为什么 gcc 不用 `.PHONY`？加速编译的效率！！！"**
> 因为 `test.exe` 是**真目标**（有实物文件），它需要 mtime 判断来"跳过不必要的编译"，这正是 make 的价值；而 `clean` 是**假目标**（没有实物），只能靠 `.PHONY` 强制每次都跑。

### 5.8 退出码：`echo $?`

```bash
[whb@bite-alicloud lesson9]$ echo $?
0
```

> `$?` 是**上一条命令的退出码**：**`0` 表示成功，非 0 表示失败**（不同数字代表不同错误）。
> `make` 判断依赖方法是否成功，靠的就是这个退出码——**这也是"依赖方法可以是任何 shell 命令"能成立的基础**：make 只看退出码。

> 板书的语法细节还提到 `errno`、`stdout`、`stderr` 三件套（后面 `fd` 与缓冲区那几课会展开）。

---

## 六、进度条：回车换行与缓冲区

### 6.1 两个背景知识

**背景 1：回车和换行是两个动作**

```text
\r    回车（Carriage Return）：光标回到当前行的行首（列归 0）
\n    换行（Line Feed）：光标下移一行

\r\n  才是"回到行首 + 下一行"（Windows 的换行符）
\n    在 Linux 上默认就能实现"换行"的显示效果
```

> 板书结论：**"显示方式：行刷新，`\n` 或 `\r\n`，按照行为单位刷新。"**
> 显示器是**行设备**：**数据先攒在缓冲区里，遇到换行（或手动刷新）才真正"上屏"。**

**背景 2：程序运行起来，缓冲区内部的数据会被自动刷新**

```c
printf("hello world\n");
sleep(2);
```

```c
printf("hello world");     // 注意：没有 \n
sleep(2);
```

两个版本的差别（板书追问）：

- 第一个：`\n` 触发刷新，**"hello world" 立刻出现**。
- 第二个：**"没有打印到显示器，看不到"** —— 因为数据还躺在**缓冲区**里！
- **"在我 sleep 期间，printf 函数已经执行完了！！'hello world' 我们去哪？在哪里呢？!"**
  → **答案是：在内存里的显示缓冲区（`stdout` 的缓冲区）中。**
  → 程序结束时，缓冲区被自动刷新，字符串才出现。

### 6.2 `fflush`：手动刷新

**不想用 `\n`、又想立刻看到结果，该怎么办？**

```c
#include <stdio.h>
int fflush(FILE *stream);
```

```c
fflush(stdout);        // 把 stdout 缓冲区里攒着的数据立刻送到显示器
```

标准 I/O 的三个流（板书列的 C 标准头）：

```c
#include <stdio.h>
extern FILE *_IO_stdin;    // 键盘
extern FILE *_IO_stdout;   // 显示器
extern FILE *_IO_stderr;   // 显示器（错误）
```

> 呼应上一篇的"**Linux 下一切皆文件**"：**键盘、显示器都是文件**，所以它们有文件指针 `stdin`/`stdout`/`stderr`、有缓冲区、能被 `fflush`。

### 6.3 `usleep`：微秒级睡眠

```c
#include <unistd.h>
int usleep(useconds_t usec);      // 单位：微秒（μs）
```

```c
usleep(500000);    // 0.5 秒
usleep(1000000);   // 1 秒
```

> 换算：`1s = 10⁶ μs`。
> （`sleep(1)` 是秒级，`usleep` 是微秒级——做"每一格刷新一次"的动画必须用后者或其组合。）

### 6.4 倒计时 demo（`\r` 的经典用法）

```c
#include <stdio.h>
#include <unistd.h>

int main()
{
    int cnt = 10;
    while (cnt >= 0)
    {
        printf("%-2d\r", cnt);     // \r 回到行首，覆盖上一次的输出
        fflush(stdout);            // 立刻刷新，不然看不到
        sleep(1);
        cnt--;
    }
    printf("\n");
    return 0;
}
```

运行效果（同一行从 10 变到 0）：

```text
10 9 8 7 6 5 4 3 2 1 0
```

> 两个要点：
> 1. **`\r` 让光标回到行首**，下一次 `printf` 就"覆盖"了上一次的字符 → 视觉上"原地变化"。
> 2. **必须 `fflush(stdout)`**：没有 `\n` 就不会自动刷新，不加就什么都看不到。
> 3. `%-2d` 固定两位宽度——**防止数字从 2 位变 1 位时留下"残影"**（如 `10` → `9` 会留下 `90`）。

### 6.5 进度条的样子

```text
[==========                  ] [ 45% ] [ / ]
 ↑进度条主体                  ↑百分比  ↑旋转光标
```

四个要素：

| 要素 | 做法 |
| --- | --- |
| 进度条主体 | 用一个字符数组，按比例填充 `=` 和空格 |
| 百分比 | `printf("%3d%%", rate)` |
| 旋转光标（转圈） | `"\|/-\\"` 四个字符轮换 |
| 原地刷新 | `\r` + `fflush(stdout)` |

> 板书的进度条课就是"`\r` + `fflush` + 循环"三件事的组合应用。
> 还附了一个网络下载场景：进度条要**按百分比显示**，`printf("%5.1f%%\r", ...)` 之类。

### 6.6 显示器的"全部都是字符"

> 板书的细节：**"当你临时显示输入一个数据的 `1456` 时候，屏幕上出现 `123456`"** ——
> **整数 → `printf` 格式化 → 字符打印；显示器输出的全部都是字符！**

这就是 `%d`/`%s`/`%f` 存在的意义：**它们负责把二进制数据转成人类可读的字符序列**。

---

## 七、git：版本控制器

### 7.1 从哪来

```text
1991  雷纳斯·托瓦兹（Linus）
        ↓ 自己写了一个版本控制器
      开源！  →  Git!     （同时代还有 SVN）
```

> 是的：**Linus 写了两个改变世界的东西——Linux 内核和 Git**。两个都是因为"现有工具不好用"而自己动手。

### 7.2 核心思想：只记录"变化"

板书的例子（两个同学协作写实验报告）：

```text
张三、李四共用一份实验报告

版本1：完整报告（初版）
版本2：实验报告，第 10 行 5 列，新增，"helloworld"
版本3：实验报告，第 15 行，新增，代码内容是 "int main"
版本4：实验报告，第三行，修改，stdio.h → stdlib.h
```

> **每一次只记录"变化部分"，而不是保存一整份新文件！**
> **→ 大幅减少存储的工作量！**

所以 git 不是"存了很多个副本"，而是**存"变化"（diff / snapshot 的差异）**。

### 7.3 三个区域与"分布式"

```text
一台机器
┌─────────────────────────────────────────────┐
│  工作区            本地 git 仓库             │
│  （你编辑的文件）    （.git 目录）            │
│      │  git add → 暂存区(index)             │
│      │              │  git commit            │
│      └──────────────┴───────────────────────│
└─────────────────────────────────────────────┘
         ↑ git pull            ↓ git push
    ┌────────────────────────────────┐
    │  remote：gitee / github        │
    │  （git server，网络通信功能）   │
    └────────────────────────────────┘
```

> **结论：git 是一个去中心化的、分布式的版本控制器**（每个机器上都有一份完整历史；gitee/github 只是"集中化的模式为主"的协作中心）。
>
> 板书的类比：
> - **"提供可以后悔的能力"** → 暂存区让你在 commit 前还能反悔
> - **"提供提交效率"** → 可以攒一批改动一起提交

### 7.4 命令行操作

```bash
[whb@bite-alicloud code]$ git --version
git version 1.8.3.1
```

**首次配置（必需，否则不能提交）**：

```bash
git config --global user.name "your name"
git config --global user.email you@example.com

# 如果之前提交过，想修正提交者信息
git commit --amend --reset-author
```

> 板书截图里那句告警很典型：
> **"您的所有凭据都已确认，请核查它们的正确性……您可以再次执行命令来确认本次提交所使用的用户身份。"**

**本地仓库初始化与日常流程**：

```bash
git init                    # 在目录里创建 .git
git status                  # 看当前状态（哪些文件改了/没被跟踪）
git add 文件名 / git add .   # 工作区 → 暂存区
git commit -m "提交日志"     # 暂存区 → 本地仓库
git log                     # 查看提交历史
```

```bash
[whb@bite-alicloud test_for_delete]$ ls -al
total 24
drwxr-xr-x 3 whb whb 4096 Aug  1 09:53 .
drwxr-xr-x 8 whb whb 4096 Aug  1 09:53 ..
-rw-r--r-- 1 whb whb    5 Aug  1 09:53 .gitignore
-rw-r--r-- 1 whb whb  870 Aug  1 09:53 README.en.md
drwxr-xr-x 8 whb whb 4096 Aug  1 09:53 .git        ← 仓库本体
```

```bash
[whb@bite-alicloud test_for_delete]$ git log
commit 0d5d0ea4e729886e2cee97edda160b7f05
Author: whb <helloworld>
Date:   Fri Aug 1 01:07:46 2025 +0800

    添加进度条代码

commit 6e6eb0b5d0aabdf19bc5ad3f0d85f7d64df7a14
Author: whb <helloworld>
Date:   Fri Aug 1 01:39:24 2025 +0800
```

> 板书重点圈出了这个十六进制串：**"版本号"——提交 ID 值，每一次 commit 都有一个，绝对不重复！**
> 这就是 git 的"内容寻址"：**ID 是内容的哈希**，所以同样的内容永远得到同样的 ID，改一个字节 ID 全变。

**两条铁律（板书的红字）**：

> - `git add` 不能乱写 —— **"你做了什么？"**（暂存是"声明我要提交哪些改动"，别把无关文件带进去）
> - `git commit -m "提交日志"` **必须得有，不能乱写！！**（历史是给人看的）

**与远端交互**：

```bash
git clone 仓库地址          # 第一次：把远端仓库完整拷到本地
git pull                   # 拉取远端的新提交并合并到本地
git push                   # 把本地提交推到远端
```

```bash
# 配置免密码登录（避免每次都输账号密码）
# 可用小要求输入账号和密码：your@email 的账号和密码！！
```

> 板书对 `git pull` 的解释最到位：
> **`git pull` 的本质：你要提交你的，必须把别人历史提交的代码先同步到本地！**
> 如果出现冲突，git 会把冲突标记写进文件，然后**"调用起你配置的文本编辑器（vim），把冲突同时呈现出来 → 让用户自己解决这个冲突！"**

**`.gitignore`**（板书列为"子问题"）：

```gitignore
*.o
*.exe
a.out
```

> 作用：**告诉 git 哪些文件不要纳入版本控制**（编译产物、临时文件、含密钥的配置）。这是 `git status` 保持干净的必备技能。

---

## 八、gdb：调试器

### 8.1 先解释一个必然踩到的坑

```bash
[whb@bite-alicloud lesson3]$ gdb mycmd
...
(no debugging symbols found)        ← 没有调试信息！
```

> **为什么默认编译出来的程序不能调试？**
> 因为 **`gcc`/`g++` 默认编译出的是 release 模式**（板书原话：**"gcc/g++ 加编译，默认模式：release 模式！！"**）
> —— 板书还追问了一句：**"为什么？"** 答案是 release 版本体积小、运行快，是给用户的产品；**调试信息（符号表、行号、变量名）要额外加进去，会显著增大体积**。

### 8.2 加上 `-g`

```bash
gcc mycmd.c -o mycmd-debuf -g         # 生成带调试信息的可执行程序
```

```bash
[whb@bite-alicloud lesson3]$ ll
-rwxr-xr-x 1 whb whb 8360 Aug  1 11:12 mycmd
-rwxr-xr-x 1 whb whb 9904 Aug  1 11:12 mycmd (调试信息!)
-rwxr-xr-x 1 whb whb 9120 Aug  1 11:19 mycmd-debuf
```

**用 `readelf` 看 `.debug_*` 段**：

```bash
[whb@bite-alicloud lesson3]$ readelf -S mycmd-debuf | grep -i debug
  [27] .debug_aranges    PROGBITS  0000000000000000 00001099 ...
  [28] .debug_info       PROGBITS  0000000000000000 00001152 ...
  [29] .debug_abbrev     PROGBITS  0000000000000000 0000117f ...
  [30] .debug_line       PROGBITS  0000000000000000 000011b2 ...
  [31] .debug_str        PROGBITS  0000000000000000 0000123b ...
```

> 板书的定义很关键：**"可执行程序，不仅仅是二进制的集合，内部是有固定格式的，这个固定的格式就叫 ELF。"**
> `readelf -S` 就是**读取格式信息的细节**——`.debug_*` 这些段就是 `-g` 加进来的调试信息。

### 8.3 `cgdb`：更好用的前端

```bash
sudo yum install -y cgdb
# 或
sudo apt install -y cgdb
```

> `cgdb` 在 gdb 基础上多了一个**上方代码窗口**（会高亮当前执行行），下面才是 gdb 命令行。
> 用法和 gdb 完全一致，只是多了"能看见代码"这个巨大优势。

### 8.4 gdb 命令表（板书全表）

| 命令 | 全称 / 作用 | 备注 |
| --- | --- | --- |
| `list` / `l` | 显示源代码 | 从上次位置开始，每次列出 10 行 |
| `list/l 函数名` | 列出指定函数的源代码 | 如 `l main` |
| `list/l 文件名:行号` | 列出指定文件指定行附近的代码 | 如 `l mycmd.c:20` |
| `r` / `run` | 从程序开始连续执行 | **相当于 VS 的 F5** |
| `n` / `next` | 单步执行，**不进入函数内部** | **相当于 F10（逐过程）** |
| `s` / `step` | 单步执行，**进入函数内部** | **相当于 F11（逐语句）** |
| `break`/`b` `行号` | 在指定行设置断点 | 如 `b 20`、`b mycmd.c:20` |
| `break`/`b` `函数名` | 在函数开头设置断点 | 如 `b main` |
| `info break` / `info b` | 查看当前所有断点信息 | |
| `delete` `断点号` | 删除指定断点 | 如 `delete 5` |
| `delete breakpoints` | 删除**所有**断点 | |
| `disable breakpoints` | 禁用所有断点 | `disable 3` 禁用 3 号 |
| `enable` | 启用断点 | |
| `finish` | 执行到当前函数返回，然后停下来 | **"运行完一个函数内的指定区域的代码"** |
| `print`/`p` `表达式` | 打印表达式的值 | 如 `p start+100`、`p i` |
| `set var 变量=值` | 修改运行时变量的值 | 如 `set var i=100` |
| `continue`/`c` | 从当前位置继续运行到下一个断点 | **相当于 F5** |

**两个关键区别**（板书专门写的）：

```text
逐语句：s / step     → 进入函数       F11
逐过程：n / next     → 把函数当成一个整体语句  F10
```

> 配合断点使用的常见套路：
> `b 函数` → `r` → `n`/`s` 走几步 → `p 变量` 看值 → `finish` 快速跳出当前函数 → `c` 继续。
> **`finish` 是最值钱的一条**：不用一行行走完整个函数，直接"跑到这个函数返回"。

### 8.5 调试的本质

板书这一页是整个 gdb 课的"魂"：

```text
        调工具！
  vim、gcc/g++、make/makefile、git、gdb

                 调试的本质？
                       ↓
   出问题了 → 1. 发现问题  2. 确定问题  3. 解决问题
                       ↓
       gdb 提供「找到问题」和「支持你分析问题」的工具！！
                       ↓
                  人在分析！
```

> **金句：工具只是工具，"找到问题、确定问题、解决问题"是人干的。**
> gdb 的价值在于：**让程序"停"下来给你看**（断点）、**让程序"慢"下来给你看**（单步）、**让你看到变量此刻的值**（`print`）——这是纯靠 `printf` 做不到的。
>
> 板书上方还画了一条软件发布链，解释"为什么用户手里的版本和你的不一样"：
>
> ```text
> boss → 产品经理 → 调研&需求 → 要人&钱 → 立项 → 项目经理
>                                              ↓
>                    开发开发项目 → 提交 → 项目经理审核 → 合并 → 联合测试
>                                              ↓
>                       测试（测试报告）→ 上线 → 推广
>
> gcc 给程序员提供的，必须是 release 版本的软件，因为体积小！
> 用户 --- 无用（给用户的就应该是 release）
> 调试的就 release 版本（测试环境也常在 release 上跑，所以线上 bug 难复现）
> ```

---

## 九、速查表与易错清单

### 9.1 工具链全景

| 环节 | 工具 | 关键命令/文件 |
| --- | --- | --- |
| 写代码 | vim | `vim 文件`、`~/.vimrc` |
| 编译 | gcc / g++ | `gcc x.c -o x`、`-E/-S/-c/-o/-g/-static` |
| 构建 | make + makefile | `make`、`make clean`、`.PHONY` |
| 版本控制 | git | `add/commit/push/pull/log/status/clone` |
| 调试 | gdb / cgdb | `b`/`r`/`n`/`s`/`p`/`finish`/`c` |
| 装这些工具 | yum / apt | `sudo yum install -y xxx` |

### 9.2 vim 速查

| 想做什么 | 命令 |
| --- | --- |
| 开始输入 | `i`（光标前）/ `a`（光标后） |
| 回到命令模式 | `Esc`（**万能出口**） |
| 保存并退出 | `:wq` / `:wq!` / `ZZ` |
| 不保存退出 | `:q!` |
| 显示行号 | `:set nu` |
| 跳到文首 / 文末 | `gg` / `G` |
| 跳到第 n 行 | `nG` 或 `vim 文件 +n` |
| 行首 / 行尾 | `^`（shift+6）/ `$`（shift+4） |
| 复制 / 粘贴 | `yy` / `p` |
| 删除字符 / 行 | `x` / `dd`（`ndd` 删 n 行） |
| 替换字符 / 进替换模式 | `r` / `R` |
| 撤销 / 反撤销 | `u` / `Ctrl + r` |
| 块注释 / 去注释 | `Ctrl+v` 选区 →（`shift+i` 输 `//` 后 `Esc`）/ `d` |
| 搜索 | `/关键字` + `n` |
| 不退出就编译运行 | `:!gcc % && ./a.out` |
| 分屏 | `:vs 文件`，`Ctrl + ww` 切换 |

### 9.3 gcc 常用选项

| 选项 | 作用 | 产物 |
| --- | --- | --- |
| `-E` | 只做预处理 | `.i` |
| `-S` | 编译到汇编 | `.s` |
| `-c` | 汇编到目标文件 | `.o` |
| `-o` | 指定输出文件名 | — |
| `-g` | **加调试信息**（gdb 必需） | 可调试的 ELF |
| `-static` | 强制静态链接 | 体积巨大的可执行文件 |
| `-Wall` | 打开所有警告 | — |

### 9.4 易错清单

1. **vim 里改坏了别急着退**：**一旦退出就无法撤销**，先按 `u`。
2. **`Makefile` 的依赖方法必须用 Tab 缩进**（空格会报 `missing separator`）。
3. **`make` 第二次不编译是正常行为**（mtime 判定），不是坏了；要强制重编就 `touch` 源文件或 `make clean`。
4. **`clean` 目标一定要 `.PHONY`**：否则目录里一旦有同名文件就永远不执行。
5. **`-g` 才有调试信息**：默认 release 模式 `gdb` 会提示 `no debugging symbols found`。
6. **`.o` 不能直接运行**（`cannot execute binary file`）：只有声明没有定义，必须先链接。
7. **`-static` 之前要确认装了静态库**，否则 `cannot find -lc`。
8. **`git commit` 必须写日志**，`git add` 不要 `add .` 把编译产物带上（配 `.gitignore`）。
9. **`git push` 前先 `git pull`**：本地的历史必须包含远端的历史，否则会被拒绝。
10. **`printf` 不带 `\n` 不会立刻显示**：要用 `\r` + `fflush(stdout)` 做原地动画。
11. **进度条数字要从 2 位到 1 位时补宽度**（`%-2d`），否则会留残影。
12. **`h j k l` 的理由是"手不离主键区"**，不是"方向键坏了"。

---

> 工具学完，下一步就是**用它们去理解系统**：`20250909_复习_进程开始` 起进入"Linux 系统"部分（进程、地址空间、文件描述符、信号、线程、网络）。
> 这条线的分工一直没变：**vim 写代码 → gcc 编译 → make 构建 → gdb 调试 → git 存档**，五个工具串起来，就是 Linux 下的完整开发闭环。
