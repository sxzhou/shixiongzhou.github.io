---
layout: post
title:  "linux运维常用命令-lsof"
date:   2017-09-23 23:03:03
categories: linux
tags: tools
author: "sxzhou"
---

## lsof
**lsof输出各列信息的意义如下**

* COMMAND：进程的名称
* PID：进程标识符
* USER：进程所有者
* FD：文件描述符，应用程序通过文件描述符识别该文件。如cwd、txt等
* TYPE：文件类型，如DIR、REG等
* DEVICE：指定磁盘的名称
* SIZE：文件的大小
* NODE：索引节点（文件在磁盘上的标识）
* NAME：打开文件的确切名称

>lsof abc.txt 显示开启文件abc.txt的进程<br>
lsof -c abc 显示abc进程现在打开的文件<br>
lsof -c -p 1234 列出进程号为1234的进程所打开的文件<br>
lsof -g gid 显示归属gid的进程情况<br>
lsof +d /usr/local/ 显示目录下被进程开启的文件<br>
lsof +D /usr/local/ 同上，但是会搜索目录下的目录，时间较长<br>
lsof -d 4 显示使用fd为4的进程<br>
lsof -i 用以显示符合条件的进程情况<br>
lsof -i[46][protocol][@hostname|hostaddr][:service|port]<br>
    &#160;&#160;&#160;&#160;46 --> IPv4 or IPv6<br>
    &#160;&#160;&#160;&#160;protocol --> TCP or UDP<br>
    &#160;&#160;&#160;&#160;hostname --> Internet host name<br>
    &#160;&#160;&#160;&#160;hostaddr --> IPv4地址<br>
    &#160;&#160;&#160;&#160;service --> /etc/service中的 service name (可以不止一个)<br>
    &#160;&#160;&#160;&#160;port --> 端口号 (可以不止一个)<br>
lsof `which httpd` //那个进程在使用apache的可执行文件<br>
lsof /etc/passwd //那个进程在占用/etc/passwd<br>
lsof /dev/hda6 //那个进程在占用hda6<br>
lsof /dev/cdrom //那个进程在占用光驱<br>
lsof -c sendmail //查看sendmail进程的文件使用情况<br>
lsof -c courier -u ^zahn //显示出那些文件被以courier打头的进程打开，但是并不属于用户zahn<br>
lsof -p 30297 //显示那些文件被pid为30297的进程打开<br>
lsof -D /tmp 显示所有在/tmp文件夹中打开的instance和文件的进程。但是symbol文件并不在列<br>
lsof -u1000 //查看uid是100的用户的进程的文件使用情况<br>
lsof -utony //查看用户tony的进程的文件使用情况<br>
lsof -u^tony //查看不是用户tony的进程的文件使用情况(^是取反的意思)<br>
lsof -i //显示所有打开的端口<br>
lsof -i:80 //显示所有打开80端口的进程<br>
lsof -i -U //显示所有打开的端口和UNIX domain文件<br>
lsof -i UDP@[url]www.akadia.com:123 //显示那些进程打开了到www.akadia.com的UDP的123(ntp)端口的链接<br>
lsof -i tcp@ohaha.ks.edu.tw:ftp -r //不断查看目前ftp连接的情况(-r，lsof会永远不断的执行，直到收到中断信号,+r，lsof会一直执行，直到没有档案被显示,缺省是15s刷新)<br>
lsof -i tcp@ohaha.ks.edu.tw:ftp -n //lsof -n 不将IP转换为hostname，缺省是不加上-n参数<br>
---
*注：ulimit -a：一个进程打开文件句柄（socket）数量限制*
