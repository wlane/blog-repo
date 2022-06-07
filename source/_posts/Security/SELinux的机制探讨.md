---
title: SELinux的机制探讨
tags:
- selinux
categories:
- security
---

参考：[SELinux User's and Administrator's Guide Red Hat Enterprise Linux 7 | Red Hat Customer Portal](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/7/html-single/selinux_users_and_administrators_guide/index)

注：本文涉及的操作均在centos7上完成，其他Linux发行版不一定完全相同

# 1.什么是SELinux

有助理解：[（翻译）SELinux 概念的直观解释 - 简书 (jianshu.com)](https://www.jianshu.com/p/115650a5bf41)

1. 原理

   任何操作系统都存在访问控制机制，整体上分为两类：

   - 自主访问控制（DAC，discretionary access control）：用户根据需要自己设置对文件或者目录的访问，标准的linux系统就使用了这种机制。它使用权限位（rwx）和访问控制列表（ACL）来控制文件和目录的访问权限；
   - 强制访问控制（MAC，mandatory access control）：一种基于标签（也称为上下文）的，由系统强制执行的访问控制机制。系统将标签和用来执行程序的进程关联起来，只有当目标标签等于进程标签，才允许进程从该对象或者更低级别的标签对象读取数据。

   由以上概念可以看出DAC有一个明显的缺点在于无法区分程序和用户，也就是说如果用户被授予访问某个文件的权限，那他运行的程序也有访问该文件的权限。这就导致恶意软件的攻击面很大。MAC则可以避免出现这些问题，SELinux就是一种MAC系统。

   SELinux（Security-Enhanced Linux）是美国NAS（国家安全局）创建的项目，可以让程序运行在其所需的最低权限上。SELinux中没有root这个概念，安全策略由管理员来定义。它定义系统中主体（subject，进程）对客体（object，文件、目录等系统资源）的访问和转换权限，然后使用一个安全策略来控制这些实体之间的交互。这些策略对用户来讲是透明的，只有系统管理员需要考虑如何指定策略。

   只有同时满足DAC和MAC，主体才能访问客体。

2. 工作模式

   因为linux上增强安全性的安全系统并不是只有SELinux一个，所以Linux创建了安全框架LSM（Linux Security Modules），它允许安全子系统以模块的方式来使用，这样在多个安全系统的情况下使用比较灵活，所以SELinux最终只是作为LSM中的一个模块来使用的。了解LSM可参考：[11th Annual USENIX Security Symposium — Technical Paper](https://www.usenix.org/legacy/events/sec02/full_papers/wright/wright_html/index.html)

   ![](https://images-pigo.oss-cn-beijing.aliyuncs.com/20220606235316.png)

   LSM在内核中提供了一套钩子函数，它位于DAC检查和访问最终资源之间。它会调用SELinux模块来进行MAC的检查，只有通过SELinux检查之后，才能真正访问物理资源。

3. 架构

   ![](https://images-pigo.oss-cn-beijing.aliyuncs.com/20220607000100.png)

   见上图，主要包含五个部分：

   - SELinux Abstraction and Hook Logic：处理和LSM交互钩子的模块；
   - Policy Enforcement Server：策略实施服务，用于组织访问控制的基本机制，包括是否允许访问以及安全上下文的管理等；
   - Access Vector Cache (AVC)：策略规则缓存；
   - System security policy database：系统安全性策略数据库；
   - selinuxfs：处理文件系统的辅助模块

   主要流程如下：

   - 当主体尝试访问某个客体时，在通过了DAC的控制后，请求访问客体对象；
   - 此时LSM截获到请求操作，它把主客体的安全上下文发给SELinux Abstraction and Hook Logic系统处理；
   - SELinux Abstraction and Hook Logic将接受的信息转发给模块Policy Enforcement Server；
   - Policy Enforcement Server与AVC通信，以确定是否允许访问；
   - 如果AVC没有缓存相应的策略，则需要访问System security policy database；
   - System security policy database找到相应的策略，并传递给Policy Enforcement Server；
   - Policy Enforcement Serve根据此策略确定是否允许访问，并将所以操作信息写入日志文件。

4. 访问控制机制

   SELinux目前提供了类型强制（TE）、基于角色（RBAC）和多级安全（MLS）三种强制访问机制组成的混合安全机制，但是主要使用的还是TE的机制。

# 2.强制访问控制机制

1. 类型强制（Type Enforcement）访问控制机制

   SELinux要求所有访问都必须要由明确授权，默认不允许任何访问，这就意味着在SELinux的世界里没有默认的超级用户。此时由管理员使用allow规则指定主体和客体类型授予访问权限，这里的allow规则其实也是上面架构中提及过的Access Vector，它主要由以下四个部分组成：

   - 源类型（Source type(s)）：通常是尝试访问的进程的域类型 ；
   - 目标类型（Target type(s)）：被进程访问的客体的类型 ；
   - 客体类别（Object class(es)）：指定允许访问的客体的类型 ；
   - 许可（Permission(s)）：象征目标类型允许源类型访问客体类型的访问种类。

   举例如下：

   allow user_t bin_t : file {read execute getattr}; 

   这句allow规则中，user_t是源类型，bin_t是目标类型，file是客体类别，表示一个普通文件，大括号中代表的是许可，是file客体类别有效许可的子集。综合起来就是说**拥有域类型user_t的进程可以读、执行或者获取具有bin_t类型的文件客体的属性**。

   除此之外，还有其它三种规则：

   - neverallow：表示不允许主体对客体执行指定的操作；
   - auditallow：表示允许操作并记录访问决策信息；
   - dontaudit：表示不记录违反规则的决策信息，且违反规则不影响运行。

2. 类型强制（Type Enforcement）的安全上下文

   安全上下文是SELinux访问控制的属性，系统中的进程和文件都会标记安全上下文，这些上下文信息被用来辅助进行访问控制，它可以看作是SELinux安全策略的描述。SElinux对系统中的部分命令做了一定的修改，通过添加“-Z”选项来显示主客体的安全上下文，比如：

   ~~~shell
   # 当前用户shell的上下文
   $ id -Z
   unconfined_u:unconfined_r:unconfined_t:s0
   # 显示进程的上下文
   $ ps -Z
   LABEL                             PID TTY          TIME CMD
   unconfined_u:unconfined_r:unconfined_t:s0 14816 pts/0 00:00:00 bash
   unconfined_u:unconfined_r:unconfined_t:s0 18132 pts/0 00:00:00 ps
   # 显示文件的上下文
   $ ls -Z test
   -rw-rw-r--. figo figo system_u:object_r:unlabeled_t:s0 test
   ~~~

   系统中一般的上下文规则如下：

   - 登录用户运行的程序的上下文由PAM模块中的pam_selinux.so决定；
   - rpm安装的文件由rpm包内的规则生成安全上下文；
   - 用户创建的文件由当前系统的安全策略中的规则来生成安全上下文；
   - cp命令后的文件会重新生成上下文；
   - mv命令后的文件上下文不变。

   综上可知，安全上下文的格式为：

   USER：ROLE：TYPE[：LEVEL[：CATEGORY]]

   - USER：用户的身份标识，常见的三种如下：

     - system_u：开机过程中系统进程的预设；
     - user_u：普通用户登录系统后的预设；
     - root：root登录后的预设。

     在当前默认的targeted安全策略中这个标识不太重要，在strict安全策略中比较重要。

   - ROLE：用来标识资源是属于用户、进程还是文件设备等，一般也是一下三种：

     - 文件、目录和设备的role：object_r；
     - 程序的role：system_r；
     - 用户的role：targeted安全策略为system_r，strict安全策略为sysadm_r、staff_r和user_r。用户的role类似系统中的GID，不同角色具备不同的的权限，用户可以具备多个role，但是同一时间内只能使用一个role。

   - TYPE：类型标识符

     - 将主机和客体划分为不用的组，给每个主体和系统中的客体都定义了一个类型，为进程运行提供最低的运行环境；
     - 如果是进程的类型，则类型也称为域；
     - 它是SELinux安全上下文中最重要的部分。

   - LEVEL和CATEGORY：定义层次和分类，只用于mls安全策略中 

     - LEVEL：代表安全等级，目前已经定义的安全等级为s0-s15，等级越来越高 。不同等级的资源需要对应级别的进程才能访问；
     - CATEGORY：代表分类，目前已经定义的分类为c0-c1023。

3. 基于角色（RBAC）的访问控制机制

   基于角色（RBAC）的访问控制是依靠类型强制（TE）建立的，在这种访问控制中，角色基于进程安全上下文中的角色标识符可以限制进程转变的类型，在这种情况下，可以定义专门的角色，只允许它转变为某种其他域类型，从而定义角色的限制。目前SELinux上并不推荐使用这种方式。

4. 多级安全（MLS）的访问控制机制

   大多数情况下，使用类型强制（TE）的控制机制已经足够，但是对于一些保密数据而言，MLS还是可以增强其安全性的。默认情况下也没有启用。

# 3.SELinux的配置

SELinux的配置文件如下：

~~~shell
$ ll /etc/sysconfig/selinux
lrwxrwxrwx. 1 root root 17 Mar  7  2019 /etc/sysconfig/selinux -> ../selinux/config
~~~

检查目前SELinux的状态：

~~~shell
$ sudo getenforce
Permissive
$ sudo sestatus
SELinux status:                 enabled
SELinuxfs mount:                /sys/fs/selinux
SELinux root directory:         /etc/selinux
Loaded policy name:             targeted
Current mode:                   permissive
Mode from config file:          permissive
Policy MLS status:              enabled
Policy deny_unknown status:     allowed
Max kernel policy version:      31
~~~

1. 打开或者关闭SELinux

   ~~~shell
   # 关闭SELINUX
   $ sudo vim /etc/sysconfig/selinux
   SELINUX=permissive
   ~~~

   这里的SELINUX有三种配置：

   - enforcing：强制执行SELinux设置；
   - permissive：不执行SELinux规则但是会提示；
   - disabled：关闭SELinux功能

   注意这里修改后需要重启操作系统生效。

   注意：在SELinux处于permissive或者enforcing状态时，可以使用setenforce命令来更新状态，不过重启失效。

2. 安全策略

   ~~~shell
   $ sudo vim /etc/sysconfig/selinux
   SELINUXTYPE=targeted
   ~~~

   这里的安全策略也有三种选择：

   - targeted：主要对系统中的服务进程进行访问控制，默认策略；
   - minimum：仅使用基本的策略规则，主要针对低内存系统或者手机设备等；
   - mls：Multi-Level Security，会对系统中所有的进程进行控制，即使执行一个简单的ls命令都会检查。

   不同版本的Linux可能策略不同，比如在较早的Linux发行版中，还有strict策略。

# 4.安全策略的配置

1. 工具

   由于在centos7上自带的selinux是最小化安装的模式，很多配置或者调试工具还需要再安装：

   - setools和setools-console：策略分析、检索，上下文管理，日志审计和监控等功能；
   - policycoreutils：提供restorecon、secon、setfiles、semodule、load_policy和setsebool等命令；
   - policycoreutils-python：提供semanage、audit2allow、audit2why和chcat等管理命令。

   ~~~shell
   $ sudo yum install -y setools setools-console policycoreutils policycoreutils-python
   ~~~

2. 