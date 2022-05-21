---
title: awk的用法
tags:
- awk
categories:
- shell
---

[Awk One-Liners Explained](http://www.catonmat.net/series/awk-one-liners-explained)

# 1.语法

可参考：[The GNU Awk User’s Guide](https://www.gnu.org/software/gawk/manual/gawk.html)

awk不仅是一个工具，还是一种DSL，类似于HTML或者SQL语言，用于解决某些特定领域内的问题。它可以处理多种输入，自定义用户函数和动态的正则表达式，功能强大。

1. 形式

   ~~~shell
   $ awk -h
   awk: option requires an argument -- h
   Usage: awk [POSIX or GNU style options] -f progfile [--] file ...
   Usage: awk [POSIX or GNU style options] [--] 'program' file ...
   POSIX options:          GNU long options: (standard)
           -f progfile             --file=progfile
           -F fs                   --field-separator=fs
           -v var=val              --assign=var=val
   ......
   ~~~

   常用命令选项：

   - -f progfile：从脚本文件中读取awk命令
   - -F fs：指定输入分隔符，可以是字符串和正则表达式
   - -v var=val：用户定义变量，将外部变量传递给awk

2. 结构

   > awk '{pattern + action}' {filenames}

   awk是由pattern和action组成，pattern表示awk查找的内容，action表示在匹配时执行的一系列命令。

   pattern可以是如下几种：

   - /正则表达式/：使用通配符的扩展集；
   - 关系表达式：使用运算符进行操作，可以是字符串或数字的比较测试；
   - 模式匹配表达式：用运算符\~（匹配）和 \~!（不匹配）；
   - BEGIN语句块、pattern语句块、END语句块：参见下面关于awk的工作原理的内容。

   action 由一个或多个命令、函数、表达式组成，之间由换行符或分号隔开，并位于大括号内，可以是如下几种，或者什么都没有（print）：

   - 变量或数组赋值；
   - 输出命令；
   - 内置函数；
   - 控制流语句。

# 2.处理流程分析

> awk 'BEGIN{ commands } pattern{ commands } END{ commands }'

使用上面常见的结构分析：

1. 首先执行BEGIN{ commands }内的语句块，注意这只会执行一次，经常用于变量初始化，头行打印一些表头信息，只会执行一次，在通过stdin读入数据前就被执行；
2. 从文件内容中读取一行，注意awk是以行为单位处理的，每读取一行使用pattern{ commands }循环处理可以理解成一个for循环，这也是最重要的部分；
3. 最后执行 END{ commands } ,也是执行一次，在所有行处理完后执行，一般用于打印一些统计结果。

比如下面的例子：

~~~shell
$ cat /etc/passwd |awk  -F ':'  'BEGIN {print "name,shell"}  {print $1","$7} END {print "blue,/bin/nosh"}'
name,shell
root,/bin/bash
bin,/sbin/nologin
。。。。。。
etcd,/sbin/nologin
blue,/bin/nosh
~~~

# 3.基本用法

内置变量：

- NF：字段总数
- NR：当前处理的行数
- FILENAME：当前文件名
- FS：字段分隔符，默认是空格和制表符
- RS：行分隔符，用于分割每一行，默认是换行符
- OFS：输出字段的分隔符，用于打印时分隔字段，默认为空格
- ORS：输出记录的分隔符，用于打印时分隔记录，默认为换行符
- OFMT：数字输出的格式，默认为％.6g

内置函数；

- tolower()：字符转为小写
- length()：返回字符串长度
- substr()：返回子字符串
- sin()：正弦
- cos()：余弦
- sqrt()：平方根
- rand()：随机数

~~~shell
# 打印字符串
$ echo 'this is a test' | awk '{print $0}'
this is a test
# 默认分割符是空格和制表符
$ echo 'this is a test' | awk '{print $3}'
a
# 获取用户名
$ awk -F ':' '{ print $1 }' /etc/passwd
root
daemon
bin
# 显示第一个和倒数第二个字段（使用内置变量）
$ awk -F ':' '{print $1, $(NF-1)}' /etc/passwd
root /root
daemon /usr/sbin
。。。。。。
# 显示第一个字段的大写字符（使用函数）
$ awk -F ':' '{print toupper($1)}' /etc/passwd
ROOT
BIN
DAEMON
。。。。。。
# 输出第一个字段是root的行的最后一个字段（条件）
$ awk -F ':' '$1 == "root" {print $NF}' /etc/passwd
/bin/bash
# 只输出包含bash的行（条件）
$ awk -F ':' '/bash/ {print $1}' /etc/passwd
root
lighthouse
figo
# 输出第一个字符大于m的第一个字段，否则输出no（if 条件）
$ awk -F ':' '{if ($1 > "m") print $1; else print "no"}' /etc/passwd
root
no
no
。。。。。。
~~~





本文参考以下链接：

https://cloud.tencent.com/developer/article/1159061

https://www.ruanyifeng.com/blog/2018/11/awk.html