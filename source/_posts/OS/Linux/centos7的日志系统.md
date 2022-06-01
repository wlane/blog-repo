---
title: centos7的日志系统
tags:
- log
categories:
- os
---

# 1.日志系统

​		在systemd管理的系统版本中，所有经systemd管理的服务，除非已经明确指定了输出的日志文件，否则都由systemd-journald服务监听/dev/log这个socket文件来获取日志，并且保存的是二进制文件，需要使用命令journalctl来查看相关信息。为保证足够高效，默认情况下保存在/run/log下，也就是说写入内存，保证高io，如果创建了/var/log/journal目录则保存在这里，具体保存大小，时间等参数由/etc/systemd/journald.conf配置文件控制。

​		由上面systemd的介绍可知默认情况下日志都保存在/run/log下，但是/run目录下的数据都是存在于内存中的，一旦服务器重启，数据就会消失，所以需要有一种持久化日志的方式，于是就采用了rsyslog来持久化日志，并且可以根据rsyslog的配置来过滤或者根据需要对不用应用做不同的日志配置。rsyslog服务启动后监听socket文件/run/systemd/journal/socket，从journald中获取配置文件中监听的日志种类，将获得的日志内容写入对应日志文件。

​		有了持久化日志之后，怎么处理持久化日志呢，不可能让它无限增长，这里就使用了logrotate来轮转日志，处理日志文件的归档以及保存天数等。

​		综上，systemd-journald+rsyslog+logrotate构成了目前主流linux上主要的日志系统。

# 2.rsyslog

参考：[RFC 3164 - The BSD Syslog Protocol (ietf.org)](https://datatracker.ietf.org/doc/html/rfc3164) 

之所以这里先介绍rsyslog，是因为rsyslog在很久以前就存在于linux系统之上，在还没有systemd的年代就已经开始使用，那时候在rsyslog启动之前，服务的日志由klogd来处理。

1. rsyslog历史

   rsyslog的发展历史如下：

   ~~~tex
   syslog --> syslog-ng --> rsyslog
   ~~~

   最早开始是syslog项目，但是它功能过于简单，只支持UDP连接，所以后来在1998年发布了syslog-ng，支持了tcp和过滤等功能，到了2004年，实现了rsyslog的最初版本，它支持RELP协议和使用内存缓存。在使用时需要注意版本的区别，因为有些linux版本上仍然保留了syslog或者syslog-ng的配置。

2. 配置文件/etc/rsyslog.conf

   这个配置文件当中包含了三个部分，模块、全局指令和规则。

   - 模块

     这部分主要是配置加载的模块信息，通过imjournal模块来读取systemd-journald的日志。以及是否打开日志服务器的功能。这里可以通过TCP或者UDP的方式来配置rsyslog，使之成为一个集中式日志服务器。

   - 全局指令

     关于工作目录，配置文件等常见信息的配置。

   - 规则

     这是最主要的部分，这里配置了哪些日志需要记录到哪里：

     ~~~shell
     *.info;mail.none;authpriv.none;cron.none                /var/log/messages
     authpriv.*                                              /var/log/secure
     mail.*                                                  -/var/log/maillog
     cron.*                                                  /var/log/cron
     *.emerg                                                 :omusrmsg:*
     uucp,news.crit                                          /var/log/spooler
     local7.*                                                /var/log/boot.log
     ~~~

     主要语法为 ：

     ~~~tex
     服务名称[.=!]日志等级 目的文件或其他设备
     ~~~

     - 服务名称

       | 相对序号 | 服务类别        | 说明                                                         |
       | -------- | --------------- | ------------------------------------------------------------ |
       | 0        | kern（kernel）  | 就是核心 （kernel） 产生的信息，大部分都是硬件检查以及核心功能的启用 |
       | 1        | user            | 在使用者层级所产生的信息                                     |
       | 2        | mail            | 只要与邮件收发有关的信息记录都属于这个；                     |
       | 3        | daemon          | 主要是系统的服务所产生的信息，例如 systemd 就是这个有关的信息！ |
       | 4        | auth            | 主要与认证/授权有关的机制，例如 login, ssh, su 等需要帐号/密码的操作； |
       | 5        | syslog          | 就是由syslog 相关服务产生的信息，其实就是 rsyslogd 这支程序本身产生的信息啊！ |
       | 6        | lpr             | 与打印相关的信息啊！                                         |
       | 7        | news            | 与新闻群组服务器有关的东西；                                 |
       | 8        | uucp            | 全名为 Unix to Unix Copy Protocol，早期用于 unix 系统间的程序数据交换； |
       | 9        | cron            | 就是例行工作调度 cron/at 等产生信息记录的地方；              |
       | 10       | authpriv        | 与 auth 类似，但记录较多帐号私人的信息，包括 pam 模块的运行等 |
       | 11       | ftp             | 与 FTP 有关的信息输出！                                      |
       | 16~23    | local0 ~ local7 | 保留给本机用户使用的一些设备号，常用于用户自定义输出         |

       注意这里面loca0 ~ local7 可能是我们配置的比较多的。

     - [.=!]

       .：代表比后面日志等级之上的都要记录

       =：代表只需要记录后面日志等级的日志

       !：代表记录除后面日志等级之外的其他等级日志

     - 日志等级

       | 等级数值 | 等级名称         | 说明                                                         |
       | -------- | ---------------- | ------------------------------------------------------------ |
       | 7        | debug            | 用来 debug时产生的信息数据；                                 |
       | 6        | info             | 仅是一些基本的信息说明而已；                                 |
       | 5        | notice           | 虽然是正常信息，但比 info 还需要被注意到的一些信息内容；     |
       | 4        | warning（warn）  | 提示信息，可能有问题，但是还不至于影响到某个 daemon 运行的信息；基本上， info, notice, warn 这三个信息都是在告知一些基本信息而已，应该还不至于造成一些系统运行困扰； |
       | 3        | err（error）     | 一些重大的错误信息，例如配置文件的某些设置值造成该服务服法启动的信息说明， 通常借由 err 的错误告知，应该可以了解到该服务无法启动的问题； |
       | 2        | crit（critical） | 比 error 还要严重的错误信息，这个错误已经很严重了；          |
       | 1        | alert            | 警告，已经很有问题的等级，比 crit 还要严重！                 |
       | 0        | emerg（panic）   | 疼痛等级，意指系统已经几乎要死机的状态！ 很严重的错误信息了。通常大概只有硬件出问题，导致整个核心无法顺利运行，就会出现这样的等级的讯息吧！ |

       除此之外，还有一个node的等级，代表忽略，不输出任何信息

     - 目的文件或其他设备

       | 类型     | 说明                 |
       | -------- | -------------------- |
       | 文件路径 | 就是日志文件的路径   |
       | 打印设备 | 打印机设备，用来打印 |
       | 用户     | 将日志显示给用户     |
       | 远程主机 | 远端日志服务器       |
       | *        | 代表在线的所有用户   |

       关于远程主机配置如下；

       ~~~shell
       # 远程主机配置
       $ sudo vi /etc/rsyslog.conf
       local0.*       @@192.168.1.100	# TCP传输
       local0.notice       @192.168.1.100  # UDP 传输
       ~~~

# 3.systemd-journald

systemd是从2010年fedora开始使用，所以最早也需要到这个时期的操作系统版本中才支持。

~~~shell
# 查看服务
$ sudo systemctl status systemd-journald
● systemd-journald.service - Journal Service
   Loaded: loaded (/usr/lib/systemd/system/systemd-journald.service; static; vendor preset: disabled)
   Active: active (running) since Sat 2022-04-30 23:34:35 CST; 1 months 1 days ago
     Docs: man:systemd-journald.service(8)
           man:journald.conf(5)
 Main PID: 387 (systemd-journal)
   Status: "Processing requests..."
    Tasks: 1
   Memory: 205.1M
   CGroup: /system.slice/systemd-journald.service
           └─387 /usr/lib/systemd/systemd-journald
。。。。。。
# 查看进程，排除包含中括号的内核部分
$ ps -ef | grep -v '\[' | head -5
UID        PID  PPID  C STIME TTY          TIME CMD
root         1     0  0 Apr30 ?        00:07:23 /usr/lib/systemd/systemd --switched-root --system --deserialize 22
root       387     1  0 Apr30 ?        00:01:01 /usr/lib/systemd/systemd-journald
root       408     1  0 Apr30 ?        00:00:00 /usr/lib/systemd/systemd-udevd
root       409     1  0 Apr30 ?        00:00:00 /usr/sbin/lvmetad -f
~~~

由以上进程可以看出systemd-journald是系统root进程systemd调用的第一个进程，进而后续所有的服务启动都可以使用它来记录信息。

1. 如何将日志写入到systemd-journald中

   使用logger命令

   ~~~shell
   # 发送命令
   $ logger -p syslog.info "test"
   # 查看journald
   $ sudo journalctl -f
   Jun 01 22:54:42 VM-12-3-centos figo[30692]: test
   ~~~

2. 保存日志文件

   由于默认情况下system-journald的日志驻留在内存，重启就会消失，所以在有些需要保留这种日志的场合，需要配置日志保存到其它指定路径。但需要考虑日志落盘带来的性能损失。

   ~~~shell
   $ sudo mkdir /var/log/journal
   $ sudo chown root:systemd-journal /var/log/journal
   $ sudo chmod 2775 /var/log/journal
   $ sudo systemctl restart systemd-journald.service
   $ ll /var/log/journal/
   drwxr-sr-x+ 2 root systemd-journal 4096 Nov 25  2021 21acf41b46a64ca4a55e93cb350a7749
   ~~~

# 4.logrotate

1. 执行

   由于轮转是一个例行动作，所以logrotate的执行由cron来控制：

   ~~~shell
   $ sudo cat /etc/cron.daily/logrotate
   #!/bin/sh
   
   /usr/sbin/logrotate -s /var/lib/logrotate/logrotate.status /etc/logrotate.conf
   EXITVALUE=$?
   if [ $EXITVALUE != 0 ]; then
       /usr/bin/logger -t logrotate "ALERT exited abnormally with [$EXITVALUE]"
   fi
   exit 0
   ~~~

   可以看出来在每天的例行任务中，logrotate都会检查自己的配置文件中相应的任务有没有完成。

2. 测试

   ~~~shell
   $ sudo cat /etc/logrotate.d/nginx
   /data/nginx/logs/access.log {
           weekly   # 每周进行一次
           missingok	# 忽略报错
           size=100M  # 文件大小大于100M开始处置
           rotate 10  # 保留10个归档
           compress  # 文件要压缩
           create 0640 www www	# 创建的文件属主
           notifempty	# 空文件不需要logrotate
           sharedscripts
           postrotate
                   kill -USR1 `cat /var/run/nginx.pid` > /dev/null || true
           endscript
   }
   ~~~

   可通过man 8 logrotate命令来查看具体的使用方式。



本文参考了以下网址：

[鸟哥的Linux私房菜：基础学习篇 第四版 | 鸟哥的 Linux 私房菜：基础学习篇 第四版 (gitbooks.io)](https://wizardforcel.gitbooks.io/vbird-linux-basic-4e/content/index.html)