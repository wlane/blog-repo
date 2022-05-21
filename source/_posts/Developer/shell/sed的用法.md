---
title: sed的用法
tags:
- sed
categories:
- shell
---

[Sed One-Liners Explained](https://catonmat.net/sed-one-liners-explained-part-one)

# 1.介绍

可参考：[sed, a stream editor (gnu.org)](http://www.gnu.org/software/sed/manual/sed.html)

sed全称Stream Editor，是非交互式的流编辑器。在处理数据之前，预先提供一组规则，sed按照此规则来一行行处理数据。处理时，把当前处理的行存储在临时缓冲区中，称为“模式空间“（pattern space），接着用sed命令处理缓冲区中的内容，处理完成后，把缓冲区的内容送往屏幕。接着处理下一行，这样不断重复，直到文件末尾。文件内容并没有改变，除非你使用重定向存储输出。

1. 语法

   ~~~shell
   $ sed --help
   Usage: sed [OPTION]... {script-only-if-no-other-script} [input-file]...
   
     -n, --quiet, --silent
                    suppress automatic printing of pattern space
     -e script, --expression=script
                    add the script to the commands to be executed
     -f script-file, --file=script-file
                    add the contents of script-file to the commands to be executed
     --follow-symlinks
                    follow symlinks when processing in place
     -i[SUFFIX], --in-place[=SUFFIX]
                    edit files in place (makes backup if SUFFIX supplied)
   ......
   If no -e, --expression, -f, or --file option is given, then the first
   non-option argument is taken as the sed script to interpret.  All
   remaining arguments are names of input files; if no input files are
   specified, then the standard input is read.
   ......
   ~~~

   常见选项含义如下：

   | 选项 | 含义                                                         |
   | ---- | ------------------------------------------------------------ |
   | -n   | 默认情况下，sed 会在所有的脚本指定执行完毕后，会自动输出处理后的内容，而该选项会屏蔽输出，需使用 print 命令来完成输出 |
   | -e   | 该选项会将其后跟的脚本命令添加到已有的命令中                 |
   | -f   | 该选项会将其后文件中的脚本命令添加到已有的命令中             |
   | -i   | 此选项会直接修改源文件                                       |
   | -r   | 支持扩展正则表达式                                           |

2. 支持的命令

   sed主要使用到的命令如下：

   | 命令 | 功能                                                       |
   | ---- | ---------------------------------------------------------- |
   | a\   | 在当前行下面插入文本                                       |
   | i\   | 在当前行上面插入文本                                       |
   | c\   | 把选定的行改为新的文本                                     |
   | d    | 删除选择的行                                               |
   | s    | 替换指定字符                                               |
   | h    | 拷贝模式空间的内容到内存中的缓冲区                         |
   | g    | 获得内存缓冲区的内容，并替代当前模式空间中的文本           |
   | l    | 列出非打印字符                                             |
   | n    | 读取下一个输入行，用下一个命令处理新的行而不是用第一个命令 |
   | p    | 打印当前行                                                 |
   | r    | 把文件内容读到模式空间                                     |
   | w    | 把模式空间的内容写到文件                                   |
   | ！   | 对没有匹配的行产生作用                                     |
   | =    | 打印当前行号                                               |

   更多内容可以使用 man sed 来查看。

   sed还支持正则表达式，基本正则表达式的元字符如下：

   | 元字符     | 功能                                                         |
   | ---------- | ------------------------------------------------------------ |
   | ^          | 匹配行开始，如：/^sed/匹配所有以sed开头的行                  |
   | $          | 匹配行结束，如：/sed$/匹配所有以sed结尾的行                  |
   | .          | 配一个非换行符的任意字符，如：/s.d/匹配s后接一个任意字符，最后是d |
   | *          | 匹配前面0个或多个字符，如：/a*sed/匹配0个或多个a后紧跟sed的行 |
   | \\+        | 匹配前面1个或多个字符                                        |
   | \\?        | 匹配前面0个或1个字符                                         |
   | []         | 匹配一个指定范围内的字符，如/[sS]ed/匹配sed和Sed             |
   | [^]        | 匹配一个不在指定范围内的字符，如：/\[^A-RT-Z\]ed/匹配不包含以A-R和T-Z的字母开头，紧跟ed的行 |
   | &          | 保存搜索字符用来替换其他字符，如：s/love/\*\*&\*\*/，love变成 \*\*love\*\* |
   | \\<        | 词首定位符，匹配单词的开始，如：/\<love/匹配包含以love开头的单词的行 |
   | \\>        | 词尾定位符，如：/love\>/匹配包含以love结尾的单词的行         |
   | \\(..\\)   | 向后引用，保存匹配的字符，如s/\(love\)able/\1rs，loveable被替换成lovers |
   | x\\{m\\}   | 重复字符x m次，如：/0\\{5\\}/匹配包含5个0的行                |
   | x\\{m,\\}  | 重复字符x，至少m次，如：/0\\{5,\\}/匹配至少有5个0的行        |
   | x\\{m,n\\} | 重复字符x，至少m次，不多于n次，如：/0\\{5,10\\}/匹配5~10个0的行 |

   还有其他一些字符以及扩展正则表达可以查看这节最开始的关于sed的参考地址。

# 2.具体用法

1. 寻址

   默认情况下，sed作用于文本数据的所有行，但是如果我们只想作用于特定的行，就需要写明address部分，如下：

   - 以数字形式指定行区间：

     ~~~shell
     # 指定第一行
     $ sed '2s/abcd/hello/' test
     abcd
     axhello
     ffghtr
     # 指定从第一行到第二行
     $ sed '1,2s/abcd/hello/' test
     hello
     axhello
     ffghtr
     ~~~

   - 用文本模式指定行区间：

     ~~~shell
     # 存在ax的行才替换
     $ sed '/ax/s/abcd/hello/' test
     abcd
     axhello
     ffghtr
     # 使用正则来匹配，替换
     $ cat test
     h1Helloh1
     h2Helloh2
     h3Helloh3
     $ cat sed.sh
     /h[0-9]/{
         s//\<&\>/1
         s//\<\/&\>/2
     }
     $ sed -f sed.sh test.txt
     <h1>Hello</h1>
     <h2>Hello</h2>
     <h3>Hello</h3>
     ~~~

2. s替换

   格式：[address]s/pattern/replacement/flags

   常用的flag标记如下：

   | 标记   | 功能                                                         |
   | ------ | ------------------------------------------------------------ |
   | n      | 1~512 之间的数字，表示指定要替换的字符串出现第几次时才进行替换，例如，一行中有 3 个 A，但用户只想替换第二个 A，那就是2 |
   | g      | 对数据中所有匹配到的内容进行替换，如果没有 g，则只会在第一次匹配成功时做替换操作。例如，一行数据中有 3 个 A，则只会替换第一个 A |
   | p      | 会打印与替换命令中指定的模式匹配的行。此标记通常与 -n 选项一起使用 |
   | w file | 将缓冲区中的内容写到指定的 file 文件中                       |
   | &      | 用正则表达式匹配的内容进行替换                               |
   | \\n    | 匹配第 n 个子串，该子串之前在 pattern 中用\ \(\\) 指定。     |
   | \\     | 转义（转义替换部分包含：&、\ 等）                            |

   ~~~shell
   # 替换每行的第二个行
   $ sed 's/h/xx/2' test
   h1Helloxx1
   h2Helloxx2
   h3Helloxx3
   # 只输出修改过的行
   $ sed -n 's/h1/xx/p' test
   xxHelloh1
   # 将修改后的内容写入到其他文件
   $ sed 's/h1/xx/w test2' test
   xxHelloh1
   h2Helloh2
   h3Helloh3
   $ cat test2
   xxHelloh1
   ~~~

3. d删除

   格式：[address]d

   ~~~shell
   # 删除从第二行到最后行所有的行
   $ sed '2,$d' test
   h1Helloh1
   # 删除符合的行之间的所有行
   $ sed '/h2/,/h3/d' test
   h1Helloh1
   ~~~

4. a和i插入命令

   格式：[address]a（或 i）\新文本内容

   ~~~shell
   # 在第二行前面插入
   $ sed '2i\xxxx' test
   h1Helloh1
   xxxx
   h2Helloh2
   h3Helloh3
   # 在第二行后面插入
   $ sed '2a\xxxx' test
   h1Helloh1
   h2Helloh2
   xxxx
   h3Helloh3
   # 插入多行
   $ sed '2a\xxxx\nyyyyy' test
   h1Helloh1
   h2Helloh2
   xxxx
   yyyyy
   h3Helloh3
   ~~~

5. c替换

   格式：[address]c\用于替换的新文本

   ~~~shell
   # 将第二行整行替换
   $ sed '2c\xxxx' test
   h1Helloh1
   xxxx
   h3Helloh3
   # 将匹配h2的行整行替换
   $ sed '/h2/c\xxxx' test
   h1Helloh1
   xxxx
   h3Helloh3
   ~~~

6. y转换

   格式：[address]y/inchars/outchars/

   换命令会对 inchars 和 outchars 值进行一对一的映射，即 inchars 中的第一个字符会被转换为 outchars 中的第一个字符，第二个字符会被转换成 outchars 中的第二个字符...这个映射过程会一直持续到处理完指定字符。如果 inchars 和 outchars 的长度不同，则 sed 会产生一条错误消息。它是唯一可以处理单个字符的sed命令。

   ~~~shell
   # 将文件中1替换成5，2替换成6，3替换成7
   $ sed 'y/123/567/' test
   h5Helloh5
   h6Helloh6
   h7Helloh7
   ~~~

7. p打印

   格式：[address]p

   ~~~shell
   # 包含2的行，先打印整行，再打印替换后的值
   $ sed -n '/2/{
   > p
   > s/Hello/test/p
   > }' test
   h2Helloh2
   h2testh2
   ~~~

8. w写入文件

   格式：[address]w filename

   ~~~shell
   # 将包含h2的行写入到文件test1
   $ sed -n '/h2/w test1' test
   $ cat test1
   h2Helloh2
   ~~~

9. r插入

   格式：[address]r filename

   r 命令用于将一个独立文件的数据插入到当前数据流的指定位置

   ~~~shell
   # 将文件test1插入到文件test第二行后面 
   $ cat test1
   xxx
   $ sed '2r test1' test
   h1Helloh1
   h2Helloh2
   xxx
   h3Helloh3
   ~~~

10. q退出

    格式：[address]q

    ~~~shell
    # 到匹配h2的行时，将Hello替换成xxxx后就直接退出，不再处理接下来的数据
    $ sed '/h2/{ s/Hello/xxxx/;q; }' test
    h1Helloh1
    h2xxxxh2
    ~~~

    

本文参考以下链接：

[Linux sed命令完全攻略（超级详细） (biancheng.net)](http://c.biancheng.net/view/4028.html)

