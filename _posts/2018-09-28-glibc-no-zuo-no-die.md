---
title: 作死与救赎小记 —— 与 glibc 多版本以及动态库的故事
layout: post
tags: os
---

# 前奏

在 RedHat7.5 上部署一个系统，要用到 32 位的 libc 环境。用 `yum install glibc.i686` 安装时出现了这样的错误

    Error:  Multilib version problems found. This often means that the root
           cause is something else and multilib version checking is just
           pointing out that there is a problem. Eg.:
    
           ...

多方排查，最终发现机器上竟然同时安装了两个 64位 glibc 版本（没人知道是哪位先辈干的）

    [root@localhost ~]# rpm -qa | grep glibc
    glibc-common-2.17-222.el7.x86_64
    glibc-2.22-62.6.2.x86_64
    glibc-2.17-222.el7.x86_64
    glibc-headers-2.17-222.el7.x86_64
    glibc-devel-2.17-222.el7.x86_64

RHEL 中 32 位版本要求与 64 位有相同的版本 ( 与 `glibc-common` 需要有相同的版本 `2.17-222` )，
所以会出上面的 `multilib` 问题（不是 32 位与 64 位冲突，而是系统认为 32 位与已经安装的某个 glibc 冲突）


<a id="org7ac82f6"></a>

# 作死

考虑这是自己刚入职的第一个正经任务，去跟领导说 “因为系统 libc 版本冲突，装不下去了”，似乎是个很丢面子的事情。
于是果断向领导轻描淡写地提了下（有个问题，正在想办法），开始作死之旅

首先，检查动态库的依赖情况

    [root@localhost tmp]# ldd `which bash`
            linux-vdso.so.1 =>  (0x00007fffdbfac000)
            libtinfo.so.5 => /lib64/libtinfo.so.5 (0x00007fcd57a75000)
            libdl.so.2 => /lib64/libdl.so.2 (0x00007fcd57871000)
            libc.so.6 => /lib64/libc.so.6 (0x00007fcd574a4000)
            /lib64/ld-linux-x86-64.so.2 (0x00007fcd57c9f000)

里面关键是 libc.so.6，检查链接，确定是 RHEL 默认版本 (`2.17`)

    [root@localhost tmp]# ls -l /lib64/libc.so.6
    lrwxrwxrwx 1 root root 12 Sep 28 01:47 /lib64/libc.so.6 -> libc-2.17.so

马上，我就开始犯第一个错误：没有检查 `glibc-2.22-62.6.2.x86_64` 的内容。
而是默认它没有与系统中的库文件冲突，认为可以放心的删除之

    $ yum remove glibc-2.22-62.6.2.x86_64
    error: Failed dependencies:
            /sbin/ldconfig is needed by (installed) pcre-8.32-17.el7.x86_64
            /sbin/ldconfig is needed by (installed) zlib-1.2.7-17.el7.x86_64
            /sbin/ldconfig is needed by (installed) xz-libs-5.2.2-1.el7.x86_64
            /sbin/ldconfig is needed by (installed) popt-1.13-16.el7.x86_64
            /sbin/ldconfig is needed by (installed) libxml2-2.9.1-6.el7_2.3.x86_64
            /sbin/ldconfig is needed by (installed) bzip2-libs-1.0.6-13.el7.x86_64
            /sbin/ldconfig is needed by (installed) readline-6.2-10.el7.x86_64
            /sbin/ldconfig is needed by (installed) libffi-3.0.13-18.el7.x86_64
            /sbin/ldconfig is needed by (installed) libgpg-error-1.12-3.el7.x86_64
            ...

系统提示 `/sbin/ldconfig` 被多个软件使用。接下来犯了致命错误（不可原谅），
认为 `ldconfig` 之类的行文件不致于影响系统，于是暴力删除

    rpm -e --nodeps glibc-2.22-62.6.2.x86_64


<a id="orgfddfb0e"></a>

# 恐慌

至此，作死完成

1.  除了 cd 等 shell 的基本操作可用，其它所有命令都会报 bad ELF interpreter，包括 `ls` 或者 `ln` 等命令
    
        bash: /usr/bin/ls: /lib64/ld-linux-x86-64.so.2: bad ELF interpreter: No such file or directory
2.  ssh 无法登录，输入用户名和密码后报 `/bin/bash: No such file or directory`
3.  scp 同样无法使用

第一时间能想到的方法有

1.  用恢复系统（比如光盘或 U 盘引导）挂载上系统盘，rpm 安装 glibc
2.  手动从其它机器上拷贝动态库文件到本系统，或者解压 glibc 的 rpm 文件

如果能直接操作磁盘上的文件，似乎也不是太大的问题。只需要把 glibc 重新安装一遍（甚至手工拷贝或链接）即可。
可这偏偏是台物理机，不知道在几千公里之外。

面对仅有的远程终端（在 tmux 中，已经无法打开新的 tab），即不能使用系统命令，又不能拷贝文件。
瞬间想到几天前风传的 SF 工程师误删库被开除的新闻，想到自己刚报道没三周就要背着包走人了，只能一头冷汗


<a id="org8c9e881"></a>

# （假装）镇定

镇定一下，吃个午饭，顺便查查网上有没有解决方法。

我的误操作的主要后果是删除了系统所有可执行程序运行时依赖的动态库，正在运行的程序（如果没有动态加载）则不受影响，
这也是为什么我还能保持远程连接，还能敲命令（虽然没一个能用）

网上有高人提到一个方法，用来解决 `/lib64/libc.so.6` 被误删的情况（似乎手残的程序员不在少数，虽然我是被之前的管理员的多版本 glibc 坑的）：
通过设置 `LD_PRELOAD` 让系统执行 ELF 文件时预加载一些动态库，然后重新建立到 libc.so.6 的链接（通常是 `/usr/lib/lib64/libc-2.17.so` 之类）

是个办法，但是我还不能确定对我的情况是否有效（因为我不是简单地删除了 `libc.so.6` ，而是删除了 glibc 整个包）。

现在只能寄希望于 rpm 在执行删除操作时先删除软链接，后删除库文件。也许删除链接，后续操作就没有执行，也许能给我剩下些什么。


<a id="org6e0baa3"></a>

# 拯救

强迫自己冷静一下（暗示自己大概不会因为这个原因被开除），回到终端前，开始我的主机拯救行动。

（以下步骤是事后根据 `history` 记录回忆，部分内容后来在 docker 里模拟，看官们不要盲目参考）


<a id="org4bde924"></a>

## 第一拨

1.  确认一下系统的 libc 库文件没被删除（但愿因删除了动态库）
2.  但是 `ls` 命令不能用
3.  幸好还有个 tmux 终端（谢天谢地），tab 补全还是可以用的（万岁！）
4.  猜测 `/lib64/libc.so.6` 的源文件应该也在 `/lib64` 目录下，应该也以 `libc` 开头
5.  果然有一下 `libc-2.17.so` 在众多 `libc` 开头的补全中
6.  试试 `ls` ，报错 `/lib64/ld-linux-x86-64.so.2: bad ELF interpreter: No such file or directory`

这里报的是 `/lib64/ld-linux-x86-64.so.2` 的错，引用官方手册 (`man ld.so`) 的说法

<p class="verse">
The programs ld.so and ld-linux.so\* find and load the shared libraries needed by a program, prepare the program to run, and then run it.<br />
</p>

`ld-linux-x86-64.so` 名字看上去是个库文件，实际上是个可执行程序（可以用 `file` 命令查看）

    ELF 64-bit LSB shared object, x86-64, version 1 (SYSV), dynamically linked, BuildID[sha1]=600742622dd65cd1d5a6c74e1385453981e24b67, not stripped

其它可执行程序的运行都是通过 `ld.so` 的。 `ld.so` 为之配置好动态库依赖环境（包括 `librt` 之类的关键到 `main` 入口的内容）


<a id="org6fc6ab6"></a>

## 第二拨

1.  尝试找到 `ld-linux-x86-64.so.2` 的源文件
2.  猜测在 `/lib64` 目录下，以 ld 开头，找到了 `/lib64/ld-2.17.so`
3.  直接运行之 `/lib64/ld-2.17.so` ，没有报错（静态链接万岁! Go 语言永远健康）
    
        [root@de7d8b59a07d lib64]# /lib64/ld-2.17.so
        Usage: ld.so [OPTION]... EXECUTABLE-FILE [ARGS-FOR-PROGRAM...]
        You have invoked `ld.so', the helper program for shared library executables.
        This program usually lives in the file `/lib/ld.so', and special directives
        in executable files using ELF shared libraries tell the system's program
        loader to load the helper program from this file.  This helper program loads
        the shared libraries needed by the program executable, prepares the program
        to run, and runs it.  You may invoke this helper program directly from the
        command line to load and run an ELF executable file; this is like executing
        that file itself, but always uses this helper program from the file you
        specified, instead of the helper program file specified in the executable
        file you run.  This is mostly of use for maintainers to test new versions
        of this helper program; chances are you did not intend to run this program.
        
          --list                list all dependencies and how they are resolved
          --verify              verify that given object really is a dynamically linked
                                object we can handle
          --inhibit-cache       Do not use /etc/ld.so.cache
          --library-path PATH   use given PATH instead of content of the environment
                                variable LD_LIBRARY_PATH
          --inhibit-rpath LIST  ignore RUNPATH and RPATH information in object names
                                in LIST
          --audit LIST          use objects named in LIST as auditors
4.  用 ld.so 来执行 `ls` 命令
    
        /lib64/ld-2.17.so /bin/ls
5.  终于开始报动态库问题了
    
        error while loading shared libraries: libc.so.6: cannot open shared object file: No such file or directory
6.  设置 `LD_PRELOAD` ，包含 `libc` 文件
    
        LD_PRELOAD=/lib64/libc-2.17.so /lib64/ld-2.17.so /bin/ls
7.  报错，缺少 `libdl.so.2` （好现象）
8.  按提示添加库，运行成功！
    
        LD_PRELOAD=/lib64/libc-2.17.so:/lib64/libdl.so /lib64/ld-2.17.so /bin/ls


<a id="org089d133"></a>

## 第三拨

到这里，安心不少，至少有命令可以用了。接下来，我需要手动恢复动态链接库

1.  每添加一个链接，就试试 `ls` 命令是否可以直接使用
    
        LD_PRELOAD=/lib64/libc-2.17.so:/lib64/libdl.so /lib64/ld-2.17.so /bin/ln -s /lib64/ld-2.17.so /lib64/ld-linux-x86-64.so.2
        LD_PRELOAD=/lib64/libc-2.17.so:/lib64/libdl.so /lib64/ld-2.17.so /bin/ln -s /lib64/libc-2.17.so /lib64/libc.so.6
        LD_PRELOAD=/lib64/libc-2.17.so:/lib64/libdl.so /lib64/ld-2.17.so /bin/ln -s /lib64/libdl.so /lib64/libdl.so.2

2.  到这里 `ls` 和 `ln` 都可以直接使用了（幸福来的很突然）
3.  接着试试 `rpm` 命令，依次添加了 `libm`, `libdl`, `libpthread`, `librt` 链接（ rpm 相对 ls 要复杂的多，还用到了 math 和 pthread 等库）
    
        ln -s /lib64/libm-2.17.so /lib64/libm.so.6
        ln -s /lib64/libdl-2.17.so /lib64/libdl.so.2
        ln -s /lib64/libpthread-2.17.so /lib64/libpthread.so.0
        ln -s /lib64/librt-2.17.so /lib64/librt.so.1
4.  修好了 rpm，可以安装 glibc 的 rpm 包了
5.  去 `/var/cache/yum` 目录下找，没找到，缓存被清除了
6.  没关系，试试用 `yum` 。按照提示建立链接
    
        ln -s /lib64/libutil-2.17.so /lib64/libutil.so.1
        ln -s /lib64/libresolv-2.17.so /lib64/libresolv.so.2
        ln -s /lib64/libcrypt-2.17.so /lib64/libcrypt.so.1
7.  `yum` 不报错了，可以来查找一下
    
        yum search glibc
8.  卡在搜索这里
9.  跳过 yum，直接从外网找到相关的 rpm 包，试试用 wget 获取 （按捺不住激动）
    
        wget http://mirror.centos.org/centos/7/os/x86_64/Packages/glibc-2.17-222.el7.x86_64.rpm
10. 果然，域名解析卡住了。跳过，从其它电脑上 `nslookup` 找到 IP 地址
11. 直接用 IP 地址再次下载 `wget http://66.241.106.180/centos/7/os/x86_64/Packages/glibc-2.17-222.el7.x86_64.rpm`
12. RPM 安装之
    
        rpm -i --force glibc-2.17-222.el7.x86_64.rpm
13. 大功造成，检查一遍
    
        yum search glibc                # 可以
        rpm -qa | grep glibc            # 正常


<a id="orgad8e68f"></a>

# 小结

到这里，总算不用担心被开除了，心可以放到肚子里了。

从作死到拯救前后不过一个小时，但期间肾上腺素分泌加大，心态变化之快，不足为外人道。
套用一句话：“真是太刺激了”

有几个问题，后面需要仔细研究研究

1.  域名解析为什么会卡住
2.  ld.so 相关知识（动态库和操作系统知识）
3.  YUM 和 rpm 包管理器的冲突解决最佳实践方式

现在是时候做个小结

1.  千万不要在不确定的情况下做操作（特别是动态库和网络）
2.  千万不要在不能访问物理机器情况下，远程做危险操作
3.  遇到事情千万要冷静
4.  书到用时方恨少（我的《程序员的自我修养》似乎已经落灰不少）
5.  作为程序员一定要了解操作系统知识
6.  莫逞强
7.  自作孽，不可活

与君共勉！（用脑过度，休息一会儿）
