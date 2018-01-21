---
layout: post
title:  "linux远程文件传输的几种方式"
date:   2017-09-24 11:36:43
categories: linux
tags: tools
author: "sxzhou"
---

## FTP
>FTP 是File Transfer Protocol（文件传输协议）的英文简称，而中文简称为“文传协议”。用于Internet上的控制文件的双向传输。同时，它也是一个应用程序（Application）。基于不同的操作系统有不同的FTP应用程序，而所有这些应用程序都遵守同一种协议以传输文件。
> 
FTP也是一个客户机/服务器系统。用户通过一个支持FTP协议的客户机程序，连接到在远程主机上的FTP服务器程序。
要使用FTP服务，必须在要部署FTP服务器,本地安装FTP客户端，才能实现文件上传下载。
## SFTP
>SFTP是Secure File Transfer Protocol的缩写，安全文件传送协议。可以为传输文件提供一种安全的网络的加密方法。sftp 与 ftp 有着几乎一样的语法和功能。SFTP 为 SSH的其中一部分，是一种传输档案至 Blogger 伺服器的安全方式。其实在SSH软件包中，已经包含了一个叫作SFTP(Secure File Transfer Protocol)的安全文件信息传输子系统，SFTP本身没有单独的守护进程，它必须使用sshd守护进程（端口号默认是22）来完成相应的连接和答复操作，所以从某种意义上来说，SFTP并不像一个服务器程序，而更像是一个客户端程序。SFTP同样是使用加密传输认证信息和传输的数据，所以，使用SFTP是非常安全的。但是，由于这种传输方式使用了加密/解密技术，所以传输效率比普通的FTP要低得多，如果您对网络安全性要求更高时，可以使用SFTP代替FTP。

SFTP是SSH的一部分，因此如果两台机器可以通过SSH通信，就可以实现SFTP文件传输，而不需要部署FTP服务器。我在工作中一般使用SSH访问线上或测试环境机器，因此对比FTP，SFTP更适合文件临时性的文件传输。
关于FTP与SFTP的原理，这两篇博客讲解得比较清晰[FTP，SFTP，FTPS区别](http://blog.csdn.net/shmilychan/article/details/51848850)，[sftp和ftp 区别、工作原理等](http://blog.csdn.net/cuker919/article/details/6403925)
## scp
>Linux scp命令用于Linux之间复制文件和目录。
>scp是 secure copy的缩写, scp是linux系统下基于ssh登陆进行安全的远程文件拷贝命令。
* 本地到远程
`scp local_file remote_username@remote_ip:remote_folder`
或者
`scp local_file remote_username@remote_ip:remote_file`
或者
`scp local_file remote_ip:remote_folder`
或者
`scp local_file remote_ip:remote_file`

* 远程到本地
`scp root@www.runoob.com:/home/root/others/music /home/space/music/1.mp3`
或者
`scp -r www.runoob.com:/home/root/others/ /home/space/music/`

[SCP详细使用说明](http://www.runoob.com/linux/linux-comm-scp.html)
## sz/rz
>sz命令是利用ZModem协议来从Linux服务器传送文件到本地，一次可以传送一个或多个文件。相对应的从本地上传文件到Linux服务器，可以使用rz命令。

-a 以文本方式传输（ascii）。
-b 以二进制方式传输（binary）。
-e 对控制字符转义（escape），这可以保证文件传输正确。
如果能够确定所传输的文件是文本格式的，使用 sz -a files
如果是二进制文件，使用 sz -be files

## python -m SimpleHTTPServer
使用python内置的SimpleHTTPServer创建http服务
服务端：
`python -m SimpleHTTPServer 20000`
这样就搭建了一个简单的http服务器，使用端口20000，客户端通过浏览器或wget即可访问了
[源码分析](http://www.jb51.net/article/87463.htm)
