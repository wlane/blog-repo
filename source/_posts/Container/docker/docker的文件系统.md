---
title: docker的文件系统
tag:
- filesystem
categories:
- docker
---

[About storage drivers | Docker Documentation](https://docs.docker.com/storage/storagedriver/)

# 1. 原理

可能都看过这张关于镜像的经典图：

![](https://images-pigo.oss-cn-beijing.aliyuncs.com/20220510210850.png)

可以看的出来，镜像的文件系统是分层的，那它的机制到底是怎样的？

1. 联合文件系统（UnionFS）

   Linux操作系统中使用mount来访问文件，在此基础上，它也支持在一个目录挂载多个目标，让多个目录在单个合并视图中显示。让我们从最早的联合文件系统unionfs开始，它一开始存在的目的是为了可以改动只读文件系统上的内容，比如CD/DVD：

   ~~~shell
   # 由于unionfs使用的年代久远，早已经不在默认的仓库内，需要使用第三方仓库安装
   $ sudo yum install -y fuse
   # 下载地址：https://centos.pkgs.org/7/okey-x86_64/fuse-unionfs-0.26-2.el7.centos.x86_64.rpm.html
   $ sudo rpm -ivh fuse-unionfs-0.26-2.el7.centos.x86_64.rpm
   # https://www.freebsd.org/cgi/man.cgi?query=unionfs&sektion=8&manpath=freebsd-release-ports
   $ tree /foo1
   /foo1
   `-- a
   
   0 directories, 1 file
   $ tree /foo2
   /foo2
   `-- b
   
   0 directories, 1 file
   $ tree /mp
   mp/
   
   0 directories, 0 files
   # 挂载
   $ sudo unionfs -o dirs=/foo1=rw:/foo2=ro /mp
   # 能够看到在/mp目录下看到另外两个目录下的文件
   $ sudo ls -l /mp
   total 0
   -rw-r--r-- 1 root root 0 May 10 22:18 a
   -rw-r--r-- 1 root root 0 May 10 22:18 b
   # 切换到root，进入mp目录，发现a可写，b不可写
   [root@VM-12-3-centos mp]# echo 111 > a
   [root@VM-12-3-centos mp]# echo 111 > b
   -bash: b: Permission denied
   # 创建一个文件c，会发现c在/foo1目录下
   [root@VM-12-3-centos mp]# touch c
   [root@VM-12-3-centos mp]# ll /foo1
   total 4
   -rw-r--r-- 1 root root 4 May 10 22:29 a
   -rw-r--r-- 1 root root 0 May 10 23:02 c
   ~~~

   由上面的实验我们可以看到，将多个目录挂载到了同一个挂载点，并且指定了哪个目录是可写，哪个目录是可读的。最终的文件系统自然会根据union的顺序形成一个堆叠的概念，一般而言，最左侧也就是最上层是可读写的，其他都是可读的。

   后来联合文件系统又有了很多的实现，导致它现在已经不是指上面这一个具体的文件系统类型，而是形成了一种专门的文件系统概念。目前Docker支持的联合文件系统如下：[Docker storage drivers | Docker Documentation](https://docs.docker.com/storage/storagedriver/select-storage-driver/)

   | Linux distribution | Recommended storage drivers | Alternative drivers                   |
   | :----------------- | :-------------------------- | :------------------------------------ |
   | Ubuntu             | overlay2                    | overlay, devicemapper, aufs, zfs, vfs |
   | Debian             | overlay2                    | overlay, devicemapper, aufs, vfs      |
   | CentOS             | overlay2                    | overlay, devicemapper, zfs, vfs       |
   | Fedora             | overlay2                    | overlay, devicemapper, zfs, vfs       |
   | SLES 15            | overlay2                    | overlay, devicemapper, vfs            |
   | RHEL               | overlay2                    | overlay, devicemapper, vfs            |

   Docker最初是从aufs开始支持的，后来发展到devicemapper，overlay，最终稳定在overlay2。目前aufs，devicemapper和overlay都已经不再建议使用。

2. 为什么要使用联合文件系统

   - 资源共享：在运行多个容器的环境里，假使这些容器运行环境都是一致的，例如跑的都是java程序，那我是不是需要每一个容器都建立一个java的运行环境？如果我们使用了unionfs的话，我们只需要保存一份相同的环境，就可以被所有的容器共享使用，而不需要每个容器都自带环境；
   - 安全隔离：所有容器对共享的环境只有可读权限，如果要写的话使用copy-on-write的机制；
   - 提高了启动速度：使用共享文件后，这部分文件和数据不需要再复制，可以变相提高启动速度。

3. overlayFS的工作机制

   相比较一开始的unionfs，后来实现的联合文件系统有了很多改进。我们以overlay的文件系统为例看下它的工作机制（与overlay2不同，但是主要只是挂载共享文件的方式不一样，基本机制相同。从Linux 3.18的内核才开始集成overlay2，所以默认centos7（3.10的内核）中我们找不到mount overlay2的命令）：

   ~~~shell
   $ tree .
   .
   |-- lower
   |   |-- a.txt
   |   `-- c.py
   |-- upper
   |   |-- a.txt
   |   `-- b.go
   `-- workdir
   
   3 directories, 4 files
   # 以overlay的方式挂载到目录
   $ sudo mount -t overlay -o lowerdir=./lower,upperdir=./upper,workdir=./workdir overlay /mnt/overlay2
   $ ll /mnt/overlay2/
   total 52
   -rw-r--r-- 1 root root     5 May 10 23:58 a.txt
   -rw-r--r-- 1 root root     8 May 10 23:58 b.sh
   -rw-r--r-- 1 root root 44445 May 10 23:58 c.txt
   # 文件名相同情况下，upper具有优先级
   $ sudo cat /mnt/overlay2/a.txt
   I'm upper
   # 对挂载目录下的文件进行修改
   $ pwd
   /mnt/overlay2
   # 修改文件
   $ sudo echo xxx | sudo tee a.txt
   $ sudo rm b.sh
   $ sudo echo yyy | sudo tee c.txt
   $ ll
   total 8
   -rw-r--r-- 1 root root 5 May 11 00:33 a.txt
   -rw-r--r-- 1 root root 4 May 11 00:36 c.py
   # 查看原始目录，lower目录的数据没有变化，但是upper目录都留下了操作的痕迹，并且增加的c.txt文件在upper目录下
   $ ll upper/ lower/
   lower/:
   total 8
   -rw-r--r-- 1 root root 6 May 11 00:30 a.txt
   -rw-r--r-- 1 root root 6 May 11 00:30 c.py
   
   upper/:
   total 8
   -rw-r--r-- 1 root root 5 May 11 00:33 a.txt
   -rw-r--r-- 1 root root 4 May 11 00:36 c.py
   # 删除lowerdir里面的文件
   $ sudo rm c.py
   $ ll upper/ lower/
   lower/:
   total 8
   -rw-r--r-- 1 root root 6 May 11 00:30 a.txt
   -rw-r--r-- 1 root root 6 May 11 00:30 c.py
   
   upper/:
   total 4
   -rw-r--r-- 1 root root    5 May 11 00:33 a.txt
   c--------- 1 root root 0, 0 May 11 00:48 c.py
   # 上面的结果是在upperdir下生成一个空白的文件
   ~~~

   我们可以看出，挂载目录下的各种操作对被挂载的目录lowerdir下的文件没有任何影响，全部操作都被upperdir目录记录了下来。这里面就是使用了写时复制的技术：如果两个调用者请求相同的资源，您可以为他们提供指向相同资源的指针，而无需复制它。只有当其中一个调用者尝试写入他们的副本时，才需要复制。也就是说一开始我们看到的都是指针指向的真正文件，但是一旦我们需要处理该文件时，才会真正复制一个文件到当前路径。

   当我们尝试修改共享文件（或只读文件）时，它首先被复制到顶部可写目录（upperdir），该目录具有比只读目录（lowerdir）更高的优先级。然后当它在可写目录中时，它可以安全地修改并且它的新内容将在合并视图中可见，因为顶层具有更高的优先级。

   我们执行的最后一个操作是删除文件。为了执行删除，在可写目录中创建一个空白文件以清除我们要删除的文件。这意味着该文件实际上并未被删除，而是隐藏在合并视图中。

# 2. docker镜像分析

![](https://images-pigo.oss-cn-beijing.aliyuncs.com/20220512230828.png)

上图是docker的结构图，我们以nginx镜像为例来分析：

~~~shell
$ sudo docker inspect nginx | jq ' .[] | .GraphDriver'
{
  "Data": {
    "LowerDir": "/var/lib/docker/overlay2/417a680c56f383ad2f6213e12ff0ed1f36f47f4e80a9f3bc5d4ca785b4e8ec68/diff:/var/lib/docker/overlay2/fd9e9d8a1630d76936c279fe1f7d9605014b3022a4db42c7614e7c979f3d6a8e/diff:/var/lib/docker/overlay2/e6e2f0db45d08b4fb76d854c3e8a6c4ff7d0d2046ebc5e4013d62320cd2ea608/diff:/var/lib/docker/overlay2/e722c30b785ace1e1f78bf9a7f6e929291b75f78450d243fdafca6dcbc0b496d/diff:/var/lib/docker/overlay2/5d107bad89b44505a6adc59204a20acd70c927ce5c6dc19b9fff6b61c1aa49a7/diff",
    "MergedDir": "/var/lib/docker/overlay2/374affc4abb56d5873052271fbdb1c90da959504881ee892f4a0013a96eed933/merged",
    "UpperDir": "/var/lib/docker/overlay2/374affc4abb56d5873052271fbdb1c90da959504881ee892f4a0013a96eed933/diff",
    "WorkDir": "/var/lib/docker/overlay2/374affc4abb56d5873052271fbdb1c90da959504881ee892f4a0013a96eed933/work"
  },
  "Name": "overlay2"
}
~~~

可以看到实际上这里的镜像一共由4层。

1. LowerDir

   只读镜像层，包含了bootfs和rootfs，以及在创建镜像的过程中根据dockerfile安装的各种文件和工具。

   - bootfs：包含boot loader和kernel，用户不会修改这个文件系统。实际上，在启动（boot）过程完成后，整个内核都会被加载进内存，此时bootfs会被卸载掉从而释放出所占用的内存。同时也可以看出，对于同样内核版本的不同的Linux发行版的bootfs都是一致的；
   - rootfs：包含典型的目录结构，包括/dev, /proc, /bin, /etc, /lib, /usr, 和/tmp等再加上要运行用户应用所需要的所有配置文件，二进制文件和库文件。这个文件系统在不同的 Linux 发行版中是不同的；
   - 其他：用户应用等文件。

   ![](https://images-pigo.oss-cn-beijing.aliyuncs.com/20220512232631.png)

   以上过程就是一个完整的构建lowerdir的过程，实际上这就是一个build image的过程。

2. UpperDir

   这是可读可写的一层，启动容器后在lowerdir上一层自动创建，所有对数据的操作都在这一层发生。

3. MergedDir

   联合挂载层，它是展现出来的合并视图，优先显示container层的文件。

4. WorkDir

   用于准备合并视图的工作目录。

接下来我们运行这个镜像：

~~~shell
$ sudo docker run -d nginx
$ sudo docker inspect 30f | jq .[0].GraphDriver.Data
{
  "LowerDir": "/var/lib/docker/overlay2/97ccf4c327ca9a840929060a3f13063479b320772fc3d43a9c0bb027747115c4-init/diff:/var/lib/docker/overlay2/374affc4abb56d5873052271fbdb1c90da959504881ee892f4a0013a96eed933/diff:/var/lib/docker/overlay2/417a680c56f383ad2f6213e12ff0ed1f36f47f4e80a9f3bc5d4ca785b4e8ec68/diff:/var/lib/docker/overlay2/fd9e9d8a1630d76936c279fe1f7d9605014b3022a4db42c7614e7c979f3d6a8e/diff:/var/lib/docker/overlay2/e6e2f0db45d08b4fb76d854c3e8a6c4ff7d0d2046ebc5e4013d62320cd2ea608/diff:/var/lib/docker/overlay2/e722c30b785ace1e1f78bf9a7f6e929291b75f78450d243fdafca6dcbc0b496d/diff:/var/lib/docker/overlay2/5d107bad89b44505a6adc59204a20acd70c927ce5c6dc19b9fff6b61c1aa49a7/diff",
  "MergedDir": "/var/lib/docker/overlay2/97ccf4c327ca9a840929060a3f13063479b320772fc3d43a9c0bb027747115c4/merged",
  "UpperDir": "/var/lib/docker/overlay2/97ccf4c327ca9a840929060a3f13063479b320772fc3d43a9c0bb027747115c4/diff",
  "WorkDir": "/var/lib/docker/overlay2/97ccf4c327ca9a840929060a3f13063479b320772fc3d43a9c0bb027747115c4/work"
}
$ sudo tree /var/lib/docker/overlay2/97ccf4c327ca9a840929060a3f13063479b320772fc3d43a9c0bb027747115c4/diff
/var/lib/docker/overlay2/97ccf4c327ca9a840929060a3f13063479b320772fc3d43a9c0bb027747115c4/diff
|-- etc
|   `-- nginx
|       `-- conf.d
|           `-- default.conf
|-- run
|   `-- nginx.pid
`-- var
    `-- cache
        `-- nginx
            |-- client_temp
            |-- fastcgi_temp
            |-- proxy_temp
            |-- scgi_temp
            `-- uwsgi_temp

12 directories, 2 files
~~~

和上面还是镜像的时候相比，镜像中原来是UpperDir的id是374affc4abb56d5873052271fbdb1c90da959504881ee892f4a0013a96eed933变成了容器的LowerDir的一部分，所以它是原来镜像层中所有叠加在一起的结果，形成了容器的LowerDir。然后在UpperDir内部是可写层，也就是docker容器启动后应用生成的文件。如果去看MergedDir的话，你可以看到整个容器内的文件内容。

当我们知道上面的完整目录后，我们实际上已经可以仿照上面一节中使用mount命令来将LowerDir和UpperDir手动合并成MergedDir目录。

所以总结以上分析，最终容器是这样的：

![](https://images-pigo.oss-cn-beijing.aliyuncs.com/20220513001100.png)

其中init/diff这一层是为了docker在启动后可以生成动态的hostname等这些信息，又不至于在使用docker commit生成镜像的时候把这些内容也build到镜像里面，因为docker commit只提交可读写层。所以容器启动时会先生成一个层专门用来存放这些在启动不同容器时会变化的值，然后把它设定为只读。在它的上面才生成可读可写层。

