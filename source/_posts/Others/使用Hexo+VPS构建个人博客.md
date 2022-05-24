---
title: 使用Hexo+VPS构建个人博客
tags:
- vps
categories:
- blog

---

# 1.基本条件

国内的服务器价格上没有优势，而且还需要备案，比较麻烦，所以我们主要考虑国外的vps提供商。

1. VPS

   可以参考：https://godaddy.idcspy.com/ 或者 https://www.idcspy.com/

   综合考虑价格以及关联的域名购买，dns解析，是否需要备案等，使用了[vultr](https://www.vultr.com/)，可以使用支付宝支付，如下：

   ![](https://images-pigo.oss-cn-beijing.aliyuncs.com/20220521234430.png)

   一般1核2G的作为个人使用是足够了，因为hexo最终只是跑在http后端的一些静态文件而已。

   创建好VPS后，一定要修改ssh端口以及使用key登录，还有注意chronyd服务等。需要注意的是centos上firewalld默认打开，在配置防火墙规则的时候要特别注意。

2. 域名以及DNS解析

   也是考虑价格以及续费成本，选择了[Namesilo](https://www.namesilo.com/)，支持域名隐私保护和doamin defender，它也可以使用支付宝支付：

   ![](https://images-pigo.oss-cn-beijing.aliyuncs.com/20220521234736.png)

   其中可以使用它自带的dns解析功能：

   ![](https://images-pigo.oss-cn-beijing.aliyuncs.com/20220521172024.png)

   同时注意开启domain defender和账号的二次验证：

   ![](https://images-pigo.oss-cn-beijing.aliyuncs.com/20220521234901.png)

3. 证书

   证书使用免费的[let's encrypt](https://letsencrypt.org/)提供的免费证书，具体查看说明文档。因为VPS可以直接连接公网，所以我们这里使用[certbot](https://certbot.eff.org/)这一类在线工具来安装，免去下载上传的流程，如下图；

   <img src="https://images-pigo.oss-cn-beijing.aliyuncs.com/20220521201029.png" style="zoom:200%;" />

   我们在VPS上按照上述图片中的说明安装好certbot后，生成证书：

   ![](https://images-pigo.oss-cn-beijing.aliyuncs.com/20220521235040.png)

   请注意以上图片是在重复执行后截的图，首次执行会比上图中多出一些内容。证书生成在默认的目录下，到时配置的时候直接指定到这个路径就行。

   注意由于这个证书默认有效期是三个月，所以certbot默认安装了一个定时器自动生成新证书：

   ~~~shell
   # 以下定时器设置了每隔一段时间就启动snap.certbot.renew.service来检查证书是否过期，过期就生成新证书
   $ sudo systemctl list-timers
   NEXT                         LEFT          LAST                         PASSED    UNIT                         ACTIVATES
   Sun 2022-05-22 14:05:00 CST  4h 19min left Sun 2022-05-22 01:27:11 CST  8h ago    snap.certbot.renew.timer     snap.certbot.renew.service
   ~~~

# 2.配置

1. VPS端的配置

~~~shell
# 安装nginx
$ sudo yum install nginx -y
# 参考https://www.xwangwang.net/2022/05/21/web/nginx/Nginx%E7%9A%84%E5%AE%89%E5%85%A8%E9%85%8D%E7%BD%AE/ 配置 
$ sudo vi /etc/nginx/nginx.conf
。。。。。。
$ sudo vi /etc/nginx/conf.d/hexo.conf
server {
        listen       443 ssl http2 default_server;
        server_name  www.xwangwang.net;

        ssl_certificate "/etc/letsencrypt/live/xwangwang.net/fullchain.pem";
        ssl_certificate_key "/etc/letsencrypt/live/xwangwang.net/privkey.pem";

        location / {
        	root         /data/hexo;
            index  index.html index.htm;
        }

        error_page 404 /404.html;
            location = /40x.html {
        }

        error_page 500 502 503 504 /50x.html;
            location = /50x.html {
        }
}
# 创建git用户
$ sudo yum install git -y
$ sudo useradd git
$ echo "git ALL=(ALL)  ALL" | sudo tee -a /etc/sudoers.d/git
git ALL=(ALL)  ALL
$ sudo passwd git
。。。。。。
# 配置git免密登录
$ sudo su - git
$ ssh-keygen -t rsa
# 将自己电脑上当前用户的public key复制到此文件
$ vi .ssh/authorized_keys
。。。。。。
$ chmod 600 .ssh/authorized_keys
# 在电脑上使用git登录测试
$ ssh -p11111 git@1.1.1.1
# 回到vps,配置git的hook
$ sudo mkdir /data/hexo -p
$ sudo chown git:git /data/hexo/
$ cd ~
$ git init --bare blog.git
$ vi blog.git/hooks/post-update
#!/bin/sh
git --work-tree=/data/hexo --git-dir=/home/git/blog.git checkout -f
$ chmod +x blog.git/hooks/post-update
# 更新nginx的目录配置
$ cat /etc/nginx/conf.d/hexo.conf
	root         /data/hexo;
$ sudo systemctl restart nginx
~~~

2. 电脑本地端配置

   参考[使用Hexo+GitHub搭建个人博客](https://www.xwangwang.net/2022/05/01/Others/%E4%BD%BF%E7%94%A8Hexo+GitHub%E6%90%AD%E5%BB%BA%E4%B8%AA%E4%BA%BA%E5%8D%9A%E5%AE%A2/) 安装hexo的流程

   ~~~shell
   # 使用非标准ssh端口的连接方式
   $ cd blog
   $ vi _config.yml
   deploy:
     type: 'git'
     #repo: git@github.com:wlane/wlane.github.io.git
     repo: ssh://git@1.1.1.1:11111/home/git/blog.git
     branch: master
   # 重新发布，注意有时候在修改了主题之后需要发布二遍才能同步正确
   $ hexo d -g
   # 到VPS上查看，可以看到原来的空目录下已经有了编译后的文件
   $ cd /data/hexo/
   $ ls
   2022      about     categories  css       favicon.ico      index.html  lib   tags
   404.html  archives  CNAME       fancybox  favicon.ico.bak  js          page  Tree.png
   ~~~

3. 访问

   此时通过浏览器访问：https://doaminnames 即可看到页面

# 3.多节点同步

由于坚果云实际上不建议同步代码库，因为git仓库的同步可能会有问题发生，所以我们还需要采取其他方式。

首先在github上建立仓库，比如：我新建了一个叫blog-repo的仓库。

在第一台初始化hexo的机器上做如下操作：

~~~shell
# 同步仓库
$ git clone git@github.com:wlane/blog-repo.git
$ cd blog-repo
$ cp -r ../blog/* ./
# 注意，这里有几个隐藏文件一定要复制
$ cp -r ../blog/.deploy_git ../blog/.github ../blog/.gitignore  ./
$ git add .
$ git commit -a -m "init"
$ git push
$ cat .gitignore
.DS_Store
Thumbs.db
db.json
*.log
node_modules/
public/
.deploy*/
_multiconfig.yml% 
~~~

如上，我们能够看出不同步node_modules，public以及.deploy*目录：

![](https://images-pigo.oss-cn-beijing.aliyuncs.com/20220521230954.png)

在另外的一台机器上，我们在拉取github上的这个仓库后，需要做的是：

~~~shell
$ cd blog-repo
# 安装所有依赖
$ npm install	
# 之后需要把本地public key复制到git用户下的.ssh/authorized_keys
~~~

然后才可以像其他节点一样发布。

注意，一定要及时同步仓库，每次操作完毕，都需要push到github上。然后下次使用的时候记得要pull到最新版本。