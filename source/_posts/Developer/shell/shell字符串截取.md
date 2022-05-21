---
title: shell字符串截取
tags:
- shell
categories:
- developer
---



~~~shell
$ url=http://www.baidu.com/pic/com/a.json
~~~

# 1.匹配

~~~shell
# 最左非贪婪匹配
$ echo ${url#*/}
/www.baidu.com/pic/com/a.json
# 最左贪婪匹配
$ echo ${url##*/}
a.json
# 最右非贪婪匹配
$ echo ${url%/*}
http://www.baidu.com/pic/com
# 最右贪婪匹配
$ echo ${url%%/*}
http:
# 第一个替换匹配
$ echo ${url/com/test}
http://www.baidu.test/pic/com/a.json
# 全部替换匹配
$ echo ${url//com/test}
http://www.baidu.test/pic/test/a.json
~~~

# 2.位置截取

~~~shell
# 从左边开始 
$ echo ${url:7}
www.baidu.com/pic/com/a.json
$ echo ${url:7:3}
www
# 从右边开始
$ echo ${url:0-10}
com/a.json
$ echo ${url:0-10:3}
com
~~~

# 3.变量

~~~shell
$ a="undefined"
# 未定义或者空值，返回后面内容，此时不赋值，变量仍然为空或者未定义
$ echo ${test:-${a}}
undefined
$ echo ${url:-${a}}
http://www.baidu.com/pic/com/a.json
$ echo $test

# 未定义或者空值，返回后面内容，并赋值给变量
$ echo ${test:=${a}}
undefined
$ echo ${url:=${a}}
http://www.baidu.com/pic/com/a.json
$ echo $test
undefined
# 变量存在时才替换，但不改变变量的值
$ echo ${test:+${a}}

$ echo ${url:+${a}}
undefined
# 未定义或者空值时，输出后面的内容到stderr
$ echo ${test:?"vaule is not undefined"}
-bash: test: vaule is not undefined
$ echo ${url:?"vaule is not undefined"}
http://www.baidu.com/pic/com/a.json
~~~

