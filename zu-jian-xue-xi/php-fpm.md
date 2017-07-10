# http://www.111cn.net/sys/linux/52907.htm

# PHP-FPM进程CPU 100% 问题解决办法

**一、进程跟踪**

|  代码如下 | 复制代码 |
| :--- | :--- |
|  | \# top //找出CPU使用率高的进程PID \# strace -p PID //跟踪进程 \# ll /proc/PID/fd //查看该进程在处理哪些文件  |

将有可疑的PHP代码修改之，如：file\_get\_contents没有设置超时时间。

  
**二、内存分配**

如果进程跟踪无法找到问题所在，再从系统方面找原因，会不会有可能内存不够用？据说一个较为干净的PHP-CGI打开大概20M-30M左右的内存，决定于PHP模块开启多少。  
通过pmap指令查看PHP-CGI进程的内存使用情况  
\# pmap $\(pgrep php-cgi \|head -1\)  
按输出的结果，结合系统的内存大小，配置PHP-CGI的进程数（max\_children）。

**三、监控**

最后，还可以通过监控与自动恢复的脚本保证服务的正常运转。下面是我用到的一些脚本：  
只要一个php-cgi进程占用的内存超过 %1 就把它kill掉

|  代码如下 | 复制代码 |
| :--- | :--- |
|  | \#!/bin/sh PIDS=\`[ps](http://www.111cn.net/fw/photo.html)aux\|grep php-cgi\|grep -v grep\|awk’{if\($4&gt;=1\)print $2}’\` for PID in $PIDS do echo \`date +%F….%T\`&gt;&gt;/data/logs/phpkill.log echo $PID &gt;&gt; /data/logs/phpkill.log kill -9 $PID done |

**检测php-fpm进程**

|  代码如下 | 复制代码 |
| :--- | :--- |
|  | \#!/bin/bash netstat -tnlp \| grep “php-cgi” &gt;&gt; /dev/null \#2&&gt; /data/logs/php\_fasle.log if \[ "$?" -eq "1" \];then \#&& \[ \`netstat -tnlp \| grep 9000 \| awk '{ print $4}' \| awk -F ":" '{print $2}'\` -eq "1" \];then /usr/local/webserver/php/sbin/php-fpm start echo \`date +%F….%T\` “System memory OOM.Kill php-cgi. php-fpm service start. ” &gt;&gt; /data/logs/php\_monitor.log fi |

**通过http检测php执行**

|  代码如下 | 复制代码 |
| :--- | :--- |
|  | \#!/bin/bash status=\`curl -s –head “http://127.0.0.1:8080/chk.php” \| awk ‘/HTTP/ {print $2}’\` if \[ $status != "200" -a $status != "304" \]; then /usr/local/webserver/php/sbin/php-fpm restart echo \`date +%F….%T\` “php-fpm service restart” &gt;&gt; /data/logs/php\_monitor.log fi |

**解决方法二**

如果你常用上面复杂我们可以按下步骤来监控与重启PHP-FPM来解决此问题

PHP-FPM启动脚本/etc/init.d/php-fpm  
从PHP 5.3.3开始就已经集成了PHP-FPM所以php-fpm在PHP 5.3.2以后的版本不支持以前的php-fpm \(start\|restart\|stop\|reload\) 了，不用专门再打补丁了，只需要解开源码直接configure，关于php-fpm的编译参数有

|  代码如下 | 复制代码 |
| :--- | :--- |
|  | --enable-fpm --with-fpm-user=www --with-fpm-group=www --with-libevent-dir=libevent |

位置  
进入到php源代码目录下，然后：

|  代码如下 | 复制代码 |
| :--- | :--- |
|  | cp -f \(php -5.3.x-source-dir\)/sapi/fpm/init.d.php-fpm /etc/init.d/php-fpm |

|  代码如下 | 复制代码 |
| :--- | :--- |
|  | \#! /bin/sh  \#\#\# BEGIN INIT INFO \# Provides:          php-fpm \# Required-Start:    $remote\_fs $network \# Required-Stop:     $remote\_fs $network \# Default-Start:     2 3 4 5 \# Default-Stop:      0 1 6 \# Short-Description: starts php-fpm \# Description:       starts the PHP FastCGI Process Manager daemon \#\#\# END INIT INFO  prefix=/usr/local/php exec\_prefix=${prefix}  php\_fpm\_BIN=${exec\_prefix}/sbin/php-fpm php\_fpm\_CONF=${prefix}/etc/php-fpm.conf php\_fpm\_PID=${prefix}/var/run/php-fpm.pid  php\_opts="--fpm-config $php\_fpm\_CONF"  wait\_for\_pid \(\) {     try=0      while test $try -lt 35 ; do          case "$1" in             'created'\)             if \[ -f "$2" \] ; then                 try=''                 break             fi             ;;              'removed'\)             if \[ ! -f "$2" \] ; then                 try=''                 break             fi             ;;         esac          echo -n .         try=\`expr $try + 1\`         sleep 1      done  }  case "$1" in     start\)         echo -n "Starting php-fpm "          $php\_fpm\_BIN $php\_opts          if \[ "$?" != 0 \] ; then             echo " failed"             exit 1         fi          wait\_for\_pid created $php\_fpm\_PID          if \[ -n "$try" \] ; then             echo " failed"             exit 1         else             echo " done"         fi     ;;      stop\)         echo -n "Gracefully shutting down php-fpm "          if \[ ! -r $php\_fpm\_PID \] ; then             echo "warning, no pid file found - php-fpm is not running ?"             exit 1         fi          kill -QUIT \`cat $php\_fpm\_PID\`          wait\_for\_pid removed $php\_fpm\_PID          if \[ -n "$try" \] ; then             echo " failed. Use force-quit"             exit 1         else             echo " done"         fi     ;;      force-quit\)         echo -n "Terminating php-fpm "          if \[ ! -r $php\_fpm\_PID \] ; then             echo "warning, no pid file found - php-fpm is not running ?"             exit 1         fi          kill -TERM \`cat $php\_fpm\_PID\`          wait\_for\_pid removed $php\_fpm\_PID          if \[ -n "$try" \] ; then             echo " failed"             exit 1         else             echo " done"         fi     ;;      restart\)         $0 stop         $0 start     ;;      reload\)          echo -n "Reload service php-fpm "          if \[ ! -r $php\_fpm\_PID \] ; then             echo "warning, no pid file found - php-fpm is not running ?"             exit 1         fi          kill -USR2 \`cat $php\_fpm\_PID\`          echo " done"     ;;      \*\)         echo "Usage: $0 {start\|stop\|force-quit\|restart\|reload}"         exit 1     ;;  esac |

编辑好后保存，执行以下命令

|  代码如下 | 复制代码 |
| :--- | :--- |
|  | chmod +x /etc/init.d/php-fpm chkconfig php-fpm on \# 检查一下 chkconfig --list php-fpm php-fpm 0:off 1:off 2:on 3:on 4:on 5:on 6:off |

完成！可以使用以下命令管理php-fpm了

|  代码如下 | 复制代码 |
| :--- | :--- |
|  | service php-fpm start service php-fpm stop service php-fpm restart service php-fpm reload  /etc/init.d/php-fpm start /etc/init.d/php-fpm stop /etc/init.d/php-fpm restart /etc/init.d/php-fpm reload |

运行service php-fpm start提示出错，怎么都没找到问题所在，后来才知道要把php-fpm.conf里的PID路径前的:分号删掉

