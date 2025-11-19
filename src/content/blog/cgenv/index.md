---
title: 如何配置一个和希冀的xv6一样的环境
publishDate: 2025-10-26 04:41:00
description: 希冀平台真是石啊
tags:
  - OS
  - environment
heroImage: { src: './thumbnail.jpg', color: '#bc5fd3ff' }
language: 'Chinese'
---

## WSL安装linux

希冀太难用了，我们有必要把lab拿出来做，但是为了保证和测评机有相同的环境，我们必须复刻它。希冀平台的Linux版本是个18.04 ubuntu的魔改。但却很奇怪的使用了3.10的内核。

```bash
root@cg:-/xv6-riscv# uname -a
Linux cg 3.10.0-1160.119.1.el7.x86 64 #1 SMP Tue Jun 4 14:43:51 UTC 2024 x86 64 x86 64 x86 64 GNU/Linux
root@cg:~/xv6-riscv# uname -r
3.10.0-1160.119.1.el7.x86 64
root@cg:-/xv6-riscv# cat /etc/os-release
NAME="Ubuntu"
VERSION="18.04.4 LTS (Bionic Beaver)"
ID=ubuntu
ID LIKE debian
PRETTY NAME="Ubuntu 18.04.4 LTS"
VERSION ID="18.04"
HOME _URL="https://www.ubuntu.com/"
SUPPORT URL="https://help.ubuntu.com/"
BUG_REPORT_URL="https://bugs.launchpad.net/ubuntu/"
PRIVACY POLICY_URL="https://www.ubuntu.com/legal/terms-and-policies/privacy-policy"
VERSION CODENAME bionic
UBUNTU CODENAME bionic
```

所以我们干脆不复刻内核了，为了轻量化，我们选择新建一个Debian。

```bash
wsl --install -d Debian
```

## 编译安装qemu 5.0.0

希冀用的是一个很奇怪的qemu版本:5.0.0。
在 
```
wsl -d debian
```
中运行：

```bash
sudo apt update
// 为了解压
sudo apt install xz-utils
// 为了build
sudo apt install build-essential
sudo apt install pkg-config
sudo apt install libglib2.0-dev
sudo apt install libpixman-1-dev
```

编译qemu

```bash
wget https://download.qemu.org/qemu-5.0.0.tar.xz
tar xvJf qemu-5.0.0.tar.xz
cd qemu-5.0.0
./configure
make -j$(nproc)
```
安装qemu

```bash
sudo make install
```

用 `qemu-system-riscv64 --version` 验证,输出为安装成功：

```bash
qemu-system-riscv64 --version
QEMU emulator version 5.0.0
Copyright (c) 2003-2020 Fabrice Bellard and the QEMU Project developers
```

## 安装交叉编译器

```bash
sudo apt install gcc-riscv64-unknown-elf binutils-riscv64-unknown-elf
```

## 复制目录

在cglab中：

```bash
tar -zcvf /mnt/cgshare/lab7.tar.gz ~/xv6-riscv
```

选择“下载远程桌面内的文件”中的 lab7.tar.gz
复制到我们的 wsl debian 中解压：

```bash
tar -zxvf lab7.tar.gz
```

## 编译选项

在 `lab7` 目录下尝试 `make qemu` ,但由于我们的编译器版本比较高，报出：

```c
user/sh.c:58:1: error: infinite recursion detected [-Werror=infinite-recursion]
```

希冀上面的 GCC 8.4.0 没有这个问题是因为它的静态分析功能还不够强大。我们可以在 Makefile 中添加 `-Wno-infinite-recursion` 解决这个问题。

```Makefile
CFLAGS += -Wno-infinite-recursion
```

这样就可以成功 make qemu 了