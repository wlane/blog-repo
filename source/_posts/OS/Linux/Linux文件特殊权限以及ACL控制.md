---
title: Linux文件特殊权限以及ACL控制
tags:
- 文件权限
categories:
- Linux
---

# 1.Linux文件的特殊权限

Linux文件系统中通常使用rwx三大权限来标识文件权限，但是如果注意的话也可以看到如下特殊的权限标识：

~~~shell
$ ll /usr/bin/passwd
-rwsr-xr-x. 1 root root 27856 Apr  1  2020 /usr/bin/passwd
$ ll -d /tmp/
drwxrwxrwt. 9 root root 4096 Jun 26 23:47 /tmp/
~~~

可以看到原本在x的位置出现了s和t位，我们就以这两个文件为例说明下面的三种特殊权限。

1. SUID

   SUID全程set user ID，它的使用需要注意以下几点：

   - 只作用于二进制文件，不能作用于目录或者shell脚本；
   - 执行者对该文件本来就有x权限；
   - 权限只在执行过程中有效；
   - 执行过程中，执行者将拥有文件owner的权限。

   基于以上原理可以得知在/usr/bin/passwd这个二进制文件上，普通用户对它有x权限，所以可以对该文件设置s位，但是为什么要设置s位呢：

   ~~~shell
   $ ll /etc/shadow
   ----------. 1 root root 840 Jun  8 23:07 /etc/shadow
   ~~~

   由于我们执行passwd的命令是为了修改密码，而密码保存在这个文件，但是这个文件除超级用户root外任何用户都没有权限修改，那普通用户如何修改自己的密码呢。所以我们对/usr/bin/passwd设置s位就可以让普通用户在执行过程中暂时拥有/usr/bin/passwd文件owner root的权限，这样就可以修改密码了。

   我们可以做如下测试：

   ~~~shell
   # 普通用户无法打开/etc/shadow
   $ cat /etc/shadow
   cat: /etc/shadow: Permission denied
   # 查看cat是不是有s位
   $ ll /usr/bin/cat
   -rwxr-xr-x. 1 root root 54080 Nov 17  2020 /usr/bin/cat
   # 给cat s权限
   $ sudo chmod u+s /usr/bin/cat
   $ ll /usr/bin/cat
   -rwsr-xr-x. 1 root root 54080 Nov 17  2020 /usr/bin/cat
   # 再次查看
   $ cat /etc/shadow
   root:xxxxxxxxxxxxxx:19111:0:99999:7:::
   bin:*:17834:0:99999:7:::
   daemon:*:17834:0:99999:7:::
   adm:*:17834:0:99999:7:::
   lp:*:17834:0:99999:7:::
   。。。。。。
   ~~~

   可以看出只要设置s位，普通用户就可以拥有root的权限。以上都是用户本来就有x的权限，假如用户没有x权限呢：

   ~~~shell
   # 为避免影响系统本身，复制一个cat出来
   $ sudo cp /usr/bin/cat /tmp/
   $ ll /tmp/cat
   -rwxr-xr-x. 1 root root 54080 Jun 27 00:07 /tmp/cat
   $ echo "test" >> test
   # 测试cat是否可以正常使用
   $ /tmp/cat  test
   test 
   $ /tmp/cat /etc/shadow
   /tmp/cat: /etc/shadow: Permission denied
   # 取消x权限
   $ sudo chmod a-x /tmp/cat
   $ ll /tmp/cat
   -rw-r--r--. 1 root root 54080 Jun 27 00:07 /tmp/cat
   # 赋予s权限
   $ sudo chmod u+s /tmp/cat
   $ ll /tmp/cat
   -rwSr--r--. 1 root root 54080 Jun 27 00:07 /tmp/cat
   $ /tmp/cat  test
   -bash: /tmp/cat: Permission denied
   ~~~

   可以看出在没有x权限的情况下，给了s权限无效，并且显示的是大写的S。

2. SGID

   SGID的全称是Set Group ID，它使用的注意点如下：

   - 对普通二进制文件和目录都要有效；
   - 作用于文件时，类似于SUID，执行该文件时，获得文件属组的权限；
   - 作用于目录时，当用户有写和执行权限时，该用户就可以在该目录下创建文件，新文件属于这个目录所属的组。

   ~~~shell
   $ sudo mkdir test
   $ sudo chmod 775 test/
   $ ll -d test
   drwxrwxr-x. 2 root root 4096 Jun 27 00:34 test
   $ sudo chmod g+s test/
   $ ll -d test
   drwxrwsr-x. 2 root root 4096 Jun 27 00:34 test
   # 在当前权限下无法在目录下创建文件
   $ touch test/a
   touch: cannot touch ‘test/a’: Permission denied
   # 给执行用户赋予写和执行权限
   $ sudo chmod o+wx test/
   $ ll -d test
   drwxrwsrwx. 2 root root 4096 Jun 27 00:34 test
   # 这个时候执行用户本来就可以在目录下创建文件了、
   $ touch test/b
   $ mkdir test/a
   $ ll test/
   total 4
   drwxrwsr-x. 2 figo root 4096 Jun 27 00:37 a
   -rw-rw-r--. 1 figo root    0 Jun 27 00:38 b
   ~~~

   可以看出在设置SGID的目录下创建的目录会继承s位，并且文件属组都是目录的所在组。这一点适合于创建属于单用户但是组内其他用户也都有权限的文件。同样这里如果是S，证明该权限无效。

3. SBIT

   SBIT全称Sticky Bit，它的特点为：

   - 只作用在目录；
   - 用户要有写和执行权限；
   - 用户在此目录下创建的文件只有自己和root能删除。

   ~~~shell
   $ sudo mkdir test
   $ sudo chmod 777 test
   $ ll -d test/
   drwxrwxrwx. 2 root root 4096 Jun 27 00:51 test/
   # 创建一个文件a
   $ touch test/a
   $ ll -d test/a
   -rw-rw-r--. 2 figo figo 4096 Jun 27 00:46 test/a
   # 其他用户正常删除
   $ sudo su - xxxxx
   Last login: Mon Jun 27 00:54:26 CST 2022 on pts/0
   $ rm test/a
   rm: remove write-protected regular empty file ‘test/a’? y
   $ ll test/
   total 0
   # 加上t位
   $ sudo chmod o+t test/
   $ ll -d test/
   drwxrwxrwt. 2 root root 4096 Jun 27 00:52 test/
   # 再创建一个文件
   $ touch test/b
   $ ll test/
   total 0
   -rw-rw-r--. 1 figo figo 0 Jun 27 00:56 b
   # 使用其他用户删除
   $ sudo su - xxxxx
   Last login: Fri Apr 29 18:29:17 CST 2022 from 81.69.102.15 on pts/0
   $ rm test/b
   rm: remove write-protected regular empty file ‘test/b’? y
   rm: cannot remove ‘test/b’: Operation not permitted
   ~~~

   从上面可以看出，在没有t位的情况下，其他用户是可以删除不是自己创建的文件的，但是一旦加上t位，其他用户就无法删除文件了。同理，如果这里的t位是大写的话，那也是无效的。

注：我们在使用数字给文件赋权的时候，假如是四位数的比如：0775，第一位的0代表的就是以上三位特殊权限的和，sst大小分别位421。

# 2.ACL

可参考：[acl(5) - Linux manual page (man7.org)](https://man7.org/linux/man-pages/man5/acl.5.html)

上面的linux特殊权限仍然还是着眼于文件本身，但是考虑到用户的话，自然会想到Linux上默认的属主，属组和其他用户这种UGO区分对象的方式太粗糙了，是否有一种方式可以指定用户或组来赋予权限呢，这个就是ACL完成的工作。

ACL全称access control list，即访问控制列表。它可以提供一种更细粒度的权限控制。类似于windows上的权限分配的方式，给指定用户赋予指定权限。

1. ACL是否开启

   大多数Linux发行版都支持ACL，但是还是需要检查一下：

   ~~~shell
   $ lsblk
   NAME   MAJ:MIN RM SIZE RO TYPE MOUNTPOINT
   sr0     11:0    1  17M  0 rom
   vda    253:0    0  60G  0 disk
   └─vda1 253:1    0  60G  0 part /
   $ mount | grep vda1
   /dev/vda1 on / type ext4 (rw,relatime,seclabel,data=ordered)
   $ sudo dumpe2fs -h /dev/vda1 | grep -i 'default mount'
   dumpe2fs 1.42.9 (28-Dec-2013)
   Default mount options:    user_xattr acl
   ~~~

   以上可以看出centos7是默认支持的，假如存在默认没有开启的情况下，我们应该怎么操作呢：

   ~~~shell
   # 临时生效
   $ sudo mount -o remount, acl /
   # 永久生效
   $ sudo vi /etc/fstab
   UUID=4b499d76-769a-40a0-93dc-4a31a59add28 /                       ext4    defaults, acl        1 1
   ~~~

2. 权限设置

   - 给用户赋权

     ~~~shell
     # 使用用户figo创建文件
     [figo@VM-12-3-centos test]$ ll -d .
     drwxrwxrwx. 2 root root 4096 Jun 27 22:01 .
     [figo@VM-12-3-centos test]$ touch file
     [figo@VM-12-3-centos test]$ ll
     total 0
     -rw-rw-r--. 1 figo figo 0 Jun 27 22:03 file
     [figo@VM-12-3-centos test]$ getfacl file
     # file: file
     # owner: figo
     # group: figo
     user::rw-
     group::rw-
     other::r--
     # 用户test写入无权限
     [test@VM-12-3-centos test]$ echo test > file
     -bash: file: Permission denied
     # 给用户test赋予读写权限
     [figo@VM-12-3-centos test]$ setfacl -m u:test:rw file
     [figo@VM-12-3-centos test]$ ll file
     -rw-rw-r--+ 1 figo figo 5 Jun 27 22:07 file
     [figo@VM-12-3-centos test]$ getfacl file
     # file: file
     # owner: figo
     # group: figo
     user::rw-
     user:test:rw-
     group::rw-
     mask::rw-
     other::r--
     # 用户test再次操作
     [test@VM-12-3-centos test]$ echo test > file
     [test@VM-12-3-centos test]$ cat file
     test
     ~~~

     可以看出在给文件赋予权限后，文件权限处多出了一个+号，并且多出了user和mask两种acl权限。

     mask权限：最大有效权限，它的值取决于文件或文件夹的权限，最终acl权限是此值和命令中的权限取“与”的结果，可以使用setfacl -m m:rwx 这种方式修改默认的mask值。

   - 给组赋权

     和上面用户一样的操作，只是把setfacl命令中u换成g即可。

   - 权限继承

     这种设置可以让子文件或者子文件夹继承父文件夹的权限，是一种常见的设置方式。

     ~~~shell
     # 创建父文件夹
     [figo@VM-12-3-centos test]$ mkdir dir
     [figo@VM-12-3-centos test]$ ll -d dir/
     drwxrwxr-x. 2 figo figo 4096 Jun 27 22:15 dir/
     [figo@VM-12-3-centos test]$ getfacl dir/
     # file: dir/
     # owner: figo
     # group: figo
     user::rwx
     group::rwx
     other::r-x
     # 赋予文件夹可以继承的权限
     [figo@VM-12-3-centos test]$ setfacl -m d:u:test:rwx dir
     [figo@VM-12-3-centos test]$ ll -d dir/
     drwxrwxr-x+ 2 figo figo 4096 Jun 27 22:15 dir/
     [figo@VM-12-3-centos test]$ getfacl dir/
     # file: dir/
     # owner: figo
     # group: figo
     user::rwx
     group::rwx
     other::r-x
     default:user::rwx
     default:user:test:rwx
     default:group::rwx
     default:mask::rwx
     default:other::r-x
     ~~~

     可以看到文件夹的acl多出了一些default开头的权限，这些信息只能在文件夹上设置，然后被文件夹中的文件和子文件夹继承：

     ~~~shell
     # 创建子文件夹和文件
     [figo@VM-12-3-centos test]$ touch dir/a
     [figo@VM-12-3-centos test]$ mkdir dir/b
     [figo@VM-12-3-centos test]$ ll dir/
     total 12
     -rw-rw-r--+ 1 figo figo    0 Jun 27 22:20 a
     drwxrwxr-x+ 2 figo figo 4096 Jun 27 22:20 b
     # 
     [figo@VM-12-3-centos test]$ getfacl dir/b
     # file: dir/b
     # owner: figo
     # group: figo
     user::rwx
     user:test:rwx
     group::rwx
     mask::rwx
     other::r-x
     default:user::rwx
     default:user:test:rwx
     default:group::rwx
     default:mask::rwx
     default:other::r-x
     [figo@VM-12-3-centos test]$ getfacl dir/a
     # file: dir/a
     # owner: figo
     # group: figo
     user::rw-
     user:test:rwx                   #effective:rw-
     group::rwx                      #effective:rw-
     mask::rw-
     other::r--
     ~~~

     上面最终权限中对于文件a，可以看到多出了effective的值，这是因为文件默认是664的权限，但是它又继承了父文件夹的rwx权限，这时生效的权限仍然是只有rw，匹配mask的值。注意：这时即使给文件赋予了x的权限，也不会影响这里仍然是rw权限，因为它是一开始设置的时候确定的。

     取消acl权限：

     ~~~shell
     [figo@VM-12-3-centos test]$ setfacl -b -R dir/
     $ ll dir/a
     -rw-rw-r--. 1 figo figo 0 Jun 27 22:20 dir/a
     [figo@VM-12-3-centos test]$ getfacl dir/a
     # file: dir/a
     # owner: figo
     # group: figo
     user::rw-
     group::rw-
     other::r--
     ~~~

3. 备份和恢复权限

   cp和mv这些命令都支持acl权限，比如cp -p就可以保留acl，但是tar之类的就不会保留文件或者文件夹的acl信息，此时我们可以使用下列方式来处理：

   ~~~shell
   # 查看acl权限
   [figo@VM-12-3-centos test]$ getfacl -R dir/
   # file: dir/
   # owner: figo
   # group: figo
   user::rwx
   group::rwx
   other::r-x
   default:user::rwx
   default:group::rwx
   default:group:test:rwx
   default:mask::rwx
   default:other::r-x
   
   # file: dir//c
   # owner: figo
   # group: figo
   user::rw-
   group::rwx                      #effective:rw-
   group:test:rwx                  #effective:rw-
   mask::rw-
   other::r--
   
   # file: dir//d
   # owner: figo
   # group: figo
   user::rwx
   group::rwx
   group:test:rwx
   mask::rwx
   other::r-x
   default:user::rwx
   default:group::rwx
   default:group:test:rwx
   default:mask::rwx
   default:other::r-x
   # 保留以上信息
   [figo@VM-12-3-centos test]$ getfacl -R dir/ > dir_acl
   # 打包文件夹后再解压已经没有acl权限
   [figo@VM-12-3-centos test]$ tar zcf dir.tar dir
   [figo@VM-12-3-centos test]$ mv dir.tar ~/
   [figo@VM-12-3-centos test]$ cd
   [figo@VM-12-3-centos ~]$ tar zxf dir.tar
   [figo@VM-12-3-centos ~]$ ll dir
   total 4
   -rw-rw-r--. 1 figo figo    0 Jun 27 22:39 c
   drwxrwxr-x. 2 figo figo 4096 Jun 27 22:41 d
   # 恢复acl权限
   [figo@VM-12-3-centos ~]$ setfacl --restore /data/test/dir_acl
   # 查看acl权限
   [figo@VM-12-3-centos ~]$ getfacl -R dir
   # file: dir
   # owner: figo
   # group: figo
   user::rwx
   group::rwx
   other::r-x
   default:user::rwx
   default:group::rwx
   default:group:test:rwx
   default:mask::rwx
   default:other::r-x
   
   # file: dir/c
   # owner: figo
   # group: figo
   user::rw-
   group::rwx                      #effective:rw-
   group:test:rwx                  #effective:rw-
   mask::rw-
   other::r--
   
   # file: dir/d
   # owner: figo
   # group: figo
   user::rwx
   group::rwx
   group:test:rwx
   mask::rwx
   other::r-x
   default:user::rwx
   default:group::rwx
   default:group:test:rwx
   default:mask::rwx
   default:other::r-x
   ~~~

   可以看出使用setfacl --restore可以恢复目录所有的acl权限。

