---
title: SELinux排错
tags:
- SELinux
categories:
- security
---

参考：[HowTos/SELinux - CentOS Wiki](https://wiki.centos.org/HowTos/SELinux)

# 1. 布尔值

布尔值是一种值只能为真或假的数据类型。在SELinux中，它是功能的开关键，也就是说，SELinux中策略的的启用与否可通过布尔值来控制。

查看布尔值的方式有两种：

- 查看目录/sys/fs/selinux/booleans/下的文件；

  ~~~shell
  $ cat /sys/fs/selinux/booleans/httpd_enable_cgi
  1 1
  $ cat /sys/fs/selinux/booleans/httpd_enable_homedirs
  0 0
  ~~~

- 使用getsebool命令：

  ~~~shell
  $ getsebool -a
  abrt_anon_write --> off
  abrt_handle_event --> off
  。。。。。。
  $ getsebool -a | grep -i httpd_enable_cgi
  httpd_enable_cgi --> on
  ~~~

查看布尔值的说明：

~~~shell
$ sudo semanage boolean -l | grep httpd_enable_homedirs
httpd_enable_homedirs          (off  ,  off)  Allow httpd to enable homedirs
~~~

这个布尔值的说明貌似仍然看不太明白，接下来就看一下这个布尔值到底有哪些规则：

~~~shell
$ sudo sesearch -b httpd_enable_homedirs -AC
Found 77 semantic av rules:
DT allow httpd_sys_script_t user_home_dir_t : lnk_file { read getattr } ; [ httpd_enable_homedirs ]
DT allow httpd_user_script_t user_home_dir_t : dir { getattr search open } ; [ httpd_enable_homedirs ]
DT allow httpd_t nfs_t : lnk_file { read getattr } ; [ httpd_enable_homedirs use_nfs_home_dirs && ]
DT allow httpd_t user_home_dir_t : lnk_file { read getattr } ; [ httpd_enable_homedirs ]
DT allow httpd_sys_script_t nfs_t : file { ioctl read getattr lock open } ; [ httpd_enable_homedirs use_nfs_home_dirs && ]
DT allow httpd_sys_script_t cifs_t : lnk_file { read getattr } ; [ httpd_enable_homedirs use_samba_home_dirs && ]
DT allow httpd_t user_home_type : lnk_file { read getattr } ; [ httpd_enable_homedirs ]
。。。。。。
~~~

我们在这里可以清楚的看到当前这个布尔值到底包含了多少规则，后面中括号中是布尔值的名称，关于这里规则的含义可以参考：[SELinux的机制探讨 | repo (xwangwang.net)](https://www.xwangwang.net/2022/06/07/Security/SELinux的机制探讨/)

修改布尔值，也就是说调整SELinux的策略：

~~~SHELL
# 临时调整
$ sudo setsebool httpd_enable_homedirs on 
# 或者 
$ sudo setsebool httpd_enable_homedirs=1
$ getsebool -a | grep -i httpd_enable_homedirs
httpd_enable_homedirs --> on
# 永久生效
$ sudo setsebool -P httpd_enable_homedirs on
~~~

# 2.日志

由于违反SELinux策略的操作默认都会记录在日志中，它会在audit和messages中都保存一份，这对于查找错误原因非常有帮助。

假设我们通过浏览器访问一个由于文件类型不匹配没有权限访问的文件，产生的日志如下：

~~~shell
$ ls -lZ /usr/share/nginx/html/index.html
-rw-rw-r--. figo figo unconfined_u:object_r:user_home_t:s0 /usr/share/nginx/html/index.html
# 查看audit日志
$ sudo tail -1f /var/log/audit/audit.log
type=AVC msg=audit(1655480012.410:132688): avc:  denied  { read } for  pid=3581 comm="nginx" name="index.html" dev="vda1" ino=656342 scontext=system_u:system_r:httpd_t:s0 tcontext=unconfined_u:object_r:user_home_t:s0 tclass=file permissive=0
type=SYSCALL msg=audit(1655480012.410:132688): arch=c000003e syscall=2 success=no exit=-13 a0=55a50cd5fd6a a1=800 a2=0 a3=55a50c311110 items=0 ppid=3579 pid=3581 auid=4294967295 uid=994 gid=989 euid=994 suid=994 fsuid=994 egid=989 sgid=989 fsgid=989 tty=(none) ses=4294967295 comm="nginx" exe="/usr/sbin/nginx" subj=system_u:system_r:httpd_t:s0 key=(null)
type=PROCTITLE msg=audit(1655480012.410:132688): proctitle=6E67696E783A20776F726B65722070726F63657373
# 查看系统日志
$ sudo tail -1f /var/log/messages
Jun 17 23:33:36 VM-12-3-centos setroubleshoot: SELinux is preventing /usr/sbin/nginx from read access on the file /usr/share/nginx/html/index.html. For complete SELinux messages run: sealert -l 061fdb7b-9238-4159-9a82-9bb6e50e517f
Jun 17 23:33:36 VM-12-3-centos python: SELinux is preventing /usr/sbin/nginx from read access on the file /usr/share/nginx/html/index.html.#012#012*****  Plugin catchall_boolean (89.3 confidence) suggests   ******************#012#012If you want to allow httpd to read user content#012Then you must tell SELinux about this by enabling the 'httpd_read_user_content' boolean.#012#012Do#012setsebool -P httpd_read_user_content 1#012#012*****  Plugin catchall (11.6 confidence) suggests   **************************#012#012If you believe that nginx should be allowed read access on the index.html file by default.#012Then you should report this as a bug.#012You can generate a local policy module to allow this access.#012Do#012allow this access for now by executing:#012# ausearch -c 'nginx' --raw | audit2allow -M my-nginx#012# semodule -i my-nginx.pp#012
~~~

可以看出audit日志的显示较为复杂，而在/var/log/messages中显示的比较清楚，它甚至推荐了一种解决方案：

~~~shell
# 按照推荐操作，此步骤是自定义策略模块
$ sudo ausearch -c 'nginx' --raw | sudo audit2allow -M my-nginx
******************** IMPORTANT ***********************
To make this policy package active, execute:

semodule -i my-nginx.pp

# 此时生成了两个文件
$ ll
total 28
-rw-r--r--. 1 root root 1046 Jun 17 23:45 my-nginx.pp
-rw-r--r--. 1 root root  482 Jun 17 23:45 my-nginx.te
$ sudo semodule -i my-nginx.pp
# 可以发现这里的文件类型并没有变化，仍然是user_home_t
$ ls -lZ /usr/share/nginx/html/index.html
-rw-rw-r--. figo figo unconfined_u:object_r:user_home_t:s0 /usr/share/nginx/html/index.html
~~~

此时浏览器仍然报错，是因为上面只是解决了read权限，但是读取文件还需要open权限，继续按照提示操作，发现浏览器可以正常访问。这是为什么呢：

~~~shell
$ sudo sesearch -s httpd_t -t user_home_t -AC
   allow httpd_t user_home_t : file { read open } ;
   。。。。。。
~~~

可以看出这里多出了一条自定义的规则，允许httpd_t类型的进程read和open user_home_t类型的file。其实就是文件my-nginx.te的内容，也就是类型强制（TE）的规则：

~~~shell
$ cat my-nginx.te

module my-nginx 1.0;

require {
        type httpd_t;
        type user_home_t;
        class file { open read };
}

#============= httpd_t ==============

#!!!! The file '/usr/share/nginx/html/index.html' is mislabeled on your system.
#!!!! Fix with $ restorecon -R -v /usr/share/nginx/html/index.html
#!!!! This avc can be allowed using the boolean 'httpd_read_user_content'
allow httpd_t user_home_t:file open;

#!!!! This avc is allowed in the current policy
allow httpd_t user_home_t:file read;
~~~

同时也可以看到，自定义的规则不属于任何布尔值。

# 3.在docker中使用SELinux

先观察一下默认的类型：

~~~shell
$ sudo yum install -y docker-ce docker-ce-cli containerd.io docker-compose-plugin
$ sudo vi /etc/docker/daemon.json
{
  "exec-opts": ["native.cgroupdriver=systemd"],
  "selinux-enabled": true
}
$ sudo systemctl restart docker
$ sudo ls -lZ /etc/docker/
-rw-r--r--. root root system_u:object_r:container_config_t:s0 daemon.json
-rw-------. root root system_u:object_r:container_config_t:s0 key.json
$ sudo ls -lZ /var/lib/docker/
drwx--x--x. root root system_u:object_r:container_var_lib_t:s0 buildkit
drwx--x---. root root system_u:object_r:container_var_lib_t:s0 containers
drwx------. root root system_u:object_r:container_var_lib_t:s0 image
drwxr-x---. root root system_u:object_r:container_var_lib_t:s0 network
drwx--x---. root root system_u:object_r:container_share_t:s0 overlay2
drwx------. root root system_u:object_r:container_var_lib_t:s0 plugins
drwx------. root root system_u:object_r:container_var_lib_t:s0 runtimes
drwx------. root root system_u:object_r:container_var_lib_t:s0 swarm
drwx------. root root system_u:object_r:container_var_lib_t:s0 tmp
drwx------. root root system_u:object_r:container_var_lib_t:s0 trust
drwx-----x. root root system_u:object_r:container_var_lib_t:s0 volumes
$ ps -ef -Z |grep docker
system_u:system_r:container_runtime_t:s0 root 19853 1  1 00:19 ?       00:00:00 /usr/bin/dockerd -H fd:// --containerd=/run/containerd/containerd.sock
# 查看和container有关的规则
$ sudo semanage boolean -l | grep -i ^container
container_connect_any          (off  ,  off)  Determine whether docker can connect to all TCP ports.
container_use_cephfs           (off  ,  off)  Allow container to use cephfs
container_manage_cgroup        (off  ,  off)  Allow container to manage cgroup
# 需要打开cgroup
$ sudo setsebool -P container_manage_cgroup 1
# 检查SELinux规则
$ sudo sesearch -s container_runtime_t -t container_config_t -c file -AC
   allow container_runtime_t container_config_t : file { ioctl read write create getattr setattr lock append unlink link rename open } ;
$ sudo sesearch -s container_runtime_t -t container_var_lib_t -c dir -AC
      allow container_runtime_t container_var_lib_t : dir { ioctl read write create getattr setattr lock relabelfrom relabelto unlink link rename mounton add_name remove_name reparent search rmdir open } ;
。。。。。。
~~~

可以看出来这些都是docker安装后自带的规则。

如果挂载外部文件，需要在文件后加z：

~~~shell
$ ls -lZ a
-rw-r--r--. root root system_u:object_r:default_t:s0   a
$ sudo docker run -it -v /data/a:/data/a nginx cat /data/a
cat: /data/a: Permission denied
$ sudo docker run -it -v /data/a:/data/a:z nginx cat /data/a
hello
~~~

如果挂载外部目录，需要在目录后加Z：

~~~shell
$ sudo docker run -it -v /data/nginx/:/data/nginx nginx ls -l /data/nginx
ls: cannot open directory '/data/nginx': Permission denied
$ sudo docker run -it -v /data/nginx/:/data/nginx:Z nginx ls -l /data/nginx
total 8
drwxr-xr-x. 3 root root 4096 May  2 15:29 conf.d
-rw-r--r--. 1 root root  789 May  2 15:50 nginx.conf
~~~

