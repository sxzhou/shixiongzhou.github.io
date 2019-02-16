---
layout: post
title:  "linux命令service xxx start是如何工作的"
date:   2017-11-08 23:08:09
categories: article
tags: linux
author: "sxzhou"
---

今天测试环境mysql停掉了，按经验执行了`service mysql start`，脚本报错`mysql:unrecognized service`，之前一直用`service`这个命令，控制服务的启动停止，查看服务状态。需要研究下它是怎么执行的，`man`一下：
>service  runs a System V init script in as predictable environment as possible, removing most environment variables and with current working directory set to /.  
The SCRIPT parameter specifies a System V init script, located in /etc/init.d/SCRIPT.  The supported values of COMMAND depend on  the  invoked  script,  service passes COMMAND and OPTIONS it to the init script unmodified.  All scripts should support at least the start and stop commands.  As a special case, if COMMAND is --full-restart, the script is run twice, first with the stop command, then with the start command.
service --status-all runs all init scripts, in alphabetical order, with the status command.  

`service`命令是Redhat Linux兼容的发行版中用来控制系统服务的实用工具，它以启动、停止、重新启动和关闭系统服务，还可以显示所有系统服务的当前状态，在CentOS下，`service`命令的路径是`/sbin/service`。  
```shell
#!/bin/sh

. /etc/init.d/functions

VERSION="$(basename $0) ver. 0.91"
USAGE="Usage: $(basename $0) < option > | --status-all | \
[ service_name [ command | --full-restart ] ]"
SERVICE=
SERVICEDIR="/etc/init.d"
OPTIONS=

if [ $# -eq 0 ]; then
   echo "${USAGE}" >&2
   exit 1
fi

cd /
while [ $# -gt 0 ]; do
  case "${1}" in
    --help | -h | --h* )
       echo "${USAGE}" >&2
       exit 0
       ;;
    --version | -V )
       echo "${VERSION}" >&2
       exit 0
       ;;
    *)
       if [ -z "${SERVICE}" -a $# -eq 1 -a "${1}" = "--status-all" ]; then
          cd ${SERVICEDIR}
          for SERVICE in * ; do
            case "${SERVICE}" in
              functions | halt | killall | single| linuxconf| kudzu)
                  ;;
              *)
                if ! is_ignored_file "${SERVICE}" \
		    && [ -x "${SERVICEDIR}/${SERVICE}" ]; then
                  env -i PATH="$PATH" TERM="$TERM" "${SERVICEDIR}/${SERVICE}" status
                fi
                ;;
            esac
          done
          exit 0
       elif [ $# -eq 2 -a "${2}" = "--full-restart" ]; then
          SERVICE="${1}"
          if [ -x "${SERVICEDIR}/${SERVICE}" ]; then
            env -i PATH="$PATH" TERM="$TERM" "${SERVICEDIR}/${SERVICE}" stop
            env -i PATH="$PATH" TERM="$TERM" "${SERVICEDIR}/${SERVICE}" start
            exit $?
          fi
       elif [ -z "${SERVICE}" ]; then
         SERVICE="${1}"
       else
         OPTIONS="${OPTIONS} ${1}"
       fi
       shift
       ;;
   esac
done

if [ -f "${SERVICEDIR}/${SERVICE}" ]; then
   env -i PATH="$PATH" TERM="$TERM" "${SERVICEDIR}/${SERVICE}" ${OPTIONS}
else
   echo $"${SERVICE}: unrecognized service" >&2
   exit 1
fi

```
`service`命令真正执行的脚本放在`/etc/init.d/`这个目录下，可以看到，这个目录下放置了很多常用服务的控制脚本，`grep`一下`mysql`，发现原来机器上脚本的文件名是`mysql.server`,修改命令`service mysql.server start`，解决。  
## 扩展一下
`/etc/init.d/`是啥？有什么用？什么时候执行？ 
### linux启动过程  
1. The BIOS or a bootloader (lilo, zlilo, grub, etc) loads Linux Kernel from disk to memory, with some parameters defined in the bootloader configuration. We can see this process watching the dots that appear in the screen. Kernel file stays in the /boot directory, and is accessed only at this moment.
2. In memory, Kernel code starts to run, detecting a series of vital devices, disk partitions etc.
3. On of the last things Kernel does is to mount the / (root) filesystem, that obrigatoriamente must contain the /etc, /sbin, /bin and /lib directories.
4. Immediately behind, calls the program called init (/sbin/init) and passes the control to him.
5. The init command will read his configuration file (/etc/inittab) which defines the system runlevel, and some Shell scripts to be run.
6. These scripts will continue the setup of system's minimal infrastructure, mounting other filesystems (according to /etc/fstab), activating swap space (virtual memory), etc.
7. The last step, and most interesting for you, is the execution of the special script called /etc/rc.d/rc, which initializes the subsystems according to a directory structure under /etc/rc.d. The name rc comes from run commands.

TODO 图片  
重点关注`init`的执行过程，内核加载完成后，启动的第一个进程是`/sbin/init`，在此过程中，会读取`/etc/inittab`，确定运行级别，Linux允许为不同的场合，分配不同的开机启动程序，这就叫做"运行级别"（runlevel）。也就是说，启动时根据"运行级别"，确定要运行哪些程序。运行级别如下：  
0. 系统停机状态，系统默认运行级别不能设为0，否则不能正常启动
1. 单用户工作状态，root权限，用于系统维护，禁止远程登陆
2. 多用户状态(没有NFS)
3. 完全的多用户状态(有NFS)，登陆后进入控制台命令行模式
4. 系统未使用，保留
5. X11控制台，登陆后进入图形GUI模式
6. 系统正常关闭并重启，默认运行级别不能设为6，否则不能正常启动

我们的后台服务器运行级别是3，也就是会以3为参数，运行`/etc/rc.d/rc`，去执行`/etc/rc.d/rc3.d/`目录下的所有的rc启动脚本，`/etc/rc.d/rc3.d/`目录中的这些启动脚本实际上都是一些连接文件，而不是真正的rc启动脚本，真正的rc启动脚本实际上都是放在`/etc/rc.d/init.d/`目录下。  
`/etc/rc.d/rc3.d/`下的链接文件都是以“S/K+数字+方法名”命名的，对于以S开头的启动脚本，将以`start`参数来运行，如果以`K`开头，则将首先以`stop`为参数停止这些已经启动了的守护进程，然后再重新运行。这样做是为了保证是当`init`改变运行级别时，所有相关的守护进程都将重启。  
系统根据runlevel启动完`rcX.d`中的脚本之后,会调用`rc.local`脚本,如果你有一个脚本命令不论在3和5都想开机启动,那么就添加与此,免去`rc3.d`和`rc5.d`分别增加启动脚本工作量.  
`init`执行完成所有的脚本后，就可以建立终端，用户登录了。

### 参考
[http://www.runoob.com/linux/linux-system-boot.html](http://www.runoob.com/linux/linux-system-boot.html)  
[https://www.ibm.com/developerworks/cn/linux/l-linuxboot/index.html](https://www.ibm.com/developerworks/cn/linux/l-linuxboot/index.html)  
[https://www.ibm.com/developerworks/cn/linux/kernel/startup/index.html](https://www.ibm.com/developerworks/cn/linux/kernel/startup/index.html)  
[https://www.zhihu.com/question/20126189](https://www.zhihu.com/question/20126189)


