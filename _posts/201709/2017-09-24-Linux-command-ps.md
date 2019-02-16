---
layout: post
title:  "linux运维常用命令-ps"
date:   2017-09-24 11:36:43
categories: article
tags: linux shell
author: "sxzhou"
---

## ps -l
 **查看属于自己这次登录的PID与相关信息列出来(只与自己的bash有关)**
* F：代表这个进程标志（process flags），说明这个进程的权限，常见号码有：  若为4表示此进程的权限为root；若为1则表示此子进程仅可进行复制（fork）而无法执行（exec）；
* S：代表这个进程的状态（STAT),主要的状态有：
* R:（Running）：该进程正在运行中；
* S:（Sleep）：该进程目前正在睡眠状态（idle），但可以被唤醒（signal）；
* D：不可被唤醒的状态，通常这个进程可能在等待I/O的情况（ex>打印）；
* T：停止状态（stop），可能是在工作控制（后台暂停）或出错（traced）状态；
* Z:（Zombie）：“僵尸”状态，该进程已经终止但却无法被删除至内存外;
* UID/PID/PPID: 代表此进程被该UID所拥有的/进程的PID号码/此进程的父进程PID号码;
* C：代表CPU使用率，单位为百分比;
* PRI/NI：Priority/Nice的缩写，代表此进程被CPU所执行的优先级，数值越小代表此进程越快被CPU执行;
* ADDR/SZ/WCHAN: 都与内存有关，ADDR是kernel function,指出该进程在内存的哪个部分，如果是个running的进程，一般会显示“—”。SZ代表此进程用掉多少内存。WCHAN表示目前进程是否在运行中，同样，若为“—”表示正在运行中;
* TTY：登录者的终端位置，若为远程登录使用动态终端接口（pts/n);
* TIME: 使用CPU的时间，注意，是此进程实际花费CPU运行的时间，而不是系统时间;
* CMD: 就是command的缩写，造成此程序的触发进程的命令为何;

## ps aux
**查看系统所有进程数据(静态) USER:该进程属于哪个用户账号的**
* PID: 该进程的进程标识符；
* %CPU: 该进程使用掉的CPU资源百分比；
* %MEM：该进程所占用的物理内存百分比；
* VSZ:该进程所占用的虚拟内存量(KB);
* RSS:该进程所占用的固定的内存量(KB);
* TTY：该进程在哪个终端机上面运行，若与终端机无关则显示？另外，tty1~tty6是本机上面的登录者程序，若为pts/0等的，则表示为由网络连接进主机的进程； STAT:该进程目前的状态，状态显示与ps -l的S标识相同(R/S/T/Z);
* TIME:该进程实际使用CPU的时间；
* COMMAND:该进程的实际命令;
