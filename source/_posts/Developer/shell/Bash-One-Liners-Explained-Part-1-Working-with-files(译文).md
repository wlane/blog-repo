---
title: Bash One-Liners Explained Part 1 Working with files(译文)
tags:
- bash
categories:
- shell
---

来自：[Bash One-Liners Explained, Part I: Working with files (catonmat.net)](https://catonmat.net/bash-one-liners-explained-part-one)

# Part 1 文件处理

1. 清空文件（截断文件至0大小）

   ~~~shell
   $ > file
   ~~~

   这个单行指令使用了输出重定向字符“>”，输出的重定向会使得文件打开以便写入内容。如果文件不存在则创建，如果存在，就截断文件的大小至0。这里我们没有重定向任何内容到文件，所以它保持为空。

   如果你想使用一些字符替换文件内容，你可以这样做：

   ~~~shell
   $ echo "some string" > file
   ~~~

   这会将“some string”写入到文件中。

2. 追加字符到文件

   ~~~shell
   $ echo "foo bar baz" >> file
   ~~~

   这行命令使用了一个不同的输出重定向字符`>>`，它会追加内容到文件。如果文件不存在则创建。新加的字符串后面会跟着换行符，如果你不想在字符串后面存在换行符，给echo命令加上-n参数：

   ~~~shell
   $ echo -n "foo bar baz" >> file
   ~~~

3. 读取文件首行并且把它赋给变量

   ~~~shell
   $ read -r line < file
   ~~~

   这行指令使用了内置的bash命令read和输入重定向操作符`<`。read命令读取标准输入的一行并把它赋给变量line。-r参数表示输入的都是原始数据，也就是说反斜杠不会被忽略（它们会被原样读取），这个`<`重定向的命令打开文件准备读取，使得它成为read命令的标准输入。

   read命令会移除所有出现在特殊变量 IFS 中的字符，IFS代表了内部字段分隔符，它用来字符扩展后的分隔以及使用read内置命令把行分隔成单词。默认情况下，IFS包含空格、制表符和换行符，也就是说开头和结尾的空格和制表符都会被删除。如果你想保留它们，就需要把IFS设置为空值：

   ~~~shell
   $ IFS= read -r line < file
   ~~~

   这将针对这条命令修改IFS的值，并且保证line变量读取的第一行是包含开头和结尾空格的原始数据。

   另外一种从文件中读取第一行到变量中的方式是：

   ~~~shell
   $ line=$(head -1 file)
   ~~~

   这行命令使用了命令替换操作符`$(...)`，它在`...`部分运行命令，并返回输出。这里的情况下，命令是输出第一行内容的`head -1 file`。然后将输出赋值给变量line。使用`$(...)`等价`` `...` ``，所以你可以写成：

   ~~~shell
   $ line=`head -1 file`
   ~~~

   不过在bash中，使用$()是更好的方式，因为它更简洁和易于嵌套。

4. 依次读入文件的每一行

   ~~~shell
   $ while read -r line; do
       # do something with $line
   done < file
   ~~~

   这是唯一的一种从文件中依次读取行的方式。这种方式把read命令放在while的循环中。当read命令到达文件的结尾时，它返回一个正返回码（故障代码），同时while循环终止。

   记住read命令会删除行首和行尾的空白字符，如果你想保留它，清楚IFS变量：

   ~~~shell
   $ while IFS= read -r line; do
       # do something with $line
   done < file
   ~~~

   如果你不喜欢把`< file`放在结尾，你也可以通过管道将文件内容输送到while循环：

   ~~~shell
   $ cat file | while IFS= read -r line; do
       # do something with $line
   done
   ~~~

5. 从文件中读取随机的行并赋给变量

   ~~~shell
   $ read -r random_line < <(shuf file)
   ~~~

   仅仅使用bash无法从文件中读取随机行，所以我们需要使用外部程序来帮忙。如果你是在一台现代的Linux机器上，就会带有GNU coreutils中的shuf工具。

   这行命令使用到了进程替换操作符`<(...)`。进程替换操作符会创建一个匿名的命名管道，把进程的输出连接到命名管道的写入部分。然后bash执行进程，使用匿名的命名管道的文件名来替换整个进程替换表达式。

   当bash遇到`<(shuf file)`时，它会打开一个特殊文件`/dev/fd/n`，这里n是一个空的文件描述符，然后运行`shuf file`，将它的输出连接到`/dev/fd/n`，同时使用`/dev/fd/n`替换`<(shuf file)`，所有最终生效的命令变成了：

   ~~~shell
   $ read -r random_line < /dev/fd/n
   ~~~

   这个命令会从打乱的文件中读取第一行。

   这里有另一种在GNU sort的帮忙下实现的方式。GNU sort 使用 -R 选项来使输出随机：

   ~~~shell
   $ read -r random_line < <(sort -R file)
   ~~~

   另外一种获取随机行给变量的方式是：

   ~~~shell
   $ random_line=$(sort -R file | head -1)
   ~~~

   这里文件通过`sort -R`随机排序，然后通过`head -1`获取第一行。

6. 从文件中获取前三列（域）并赋给变量

   ~~~shell
   $ while read -r field1 field2 field3 throwaway; do
       # do something with $field1, $field2, and $field3
   done < file
   ~~~

   如果你给read命令指定多个变量，它会把一行分割成多个区域（分割是否结束取决于IFS变量，默认包含空格，制表符和换行符），它会把第一个区域赋给第一个变量，第二个区域赋给第二个变量，等等，然后它会将剩下的区域全部给最后的变量。这是为什么这里在三个变量之后有throwaway变量的原因。如果我们没有的话，如果存在大于三列的情况，则第三列包含所有剩下的值。

   有时候为了简便，只是用_来代替throwaway变量：

   ~~~shell
   $ while read -r field1 field2 field3 _; do
       # do something with $field1, $field2, and $field3
   done < file
   ~~~

   或者如果你有一个精确的只有三列的文件，那就根本就不需要它：

   ~~~shell
   $ while read -r field1 field2 field3; do
       # do something with $field1, $field2, and $field3
   done < file
   ~~~

   这里有一个例子。假设你想要的知道文件的行数，单词数和字节数。如果你运行wc命令会得到一个包含文件名作为第四列的三个数字：

   ~~~shell
   $ cat file-with-5-lines
   x 1
   x 2
   x 3
   x 4
   x 5
   
   $ wc file-with-5-lines
    5 10 20 file-with-5-lines
   ~~~

   所以这个文件有5行，10个单词和20个字符。我们可以通过read命令来得到这些值并把它赋给变量：

   ~~~shell
   $ read lines words chars _ < <(wc file-with-5-lines)
   
   $ echo $lines
   5
   $ echo $words
   10
   $ echo $chars
   20
   ~~~

   类似的，你可以使用here-strings 将字符串分割后赋给变量。假设你在$info变量中包含了字符串“20 packets in 10 seconds”，你想要导出20和10。在不久前我是这样写的：

   ~~~shell
   $ packets=$(echo $info | awk '{ print $1 }')
   $ time=$(echo $info | awk '{ print $4 }')
   ~~~

   但是在有了read的能力和bash知识之后，我们可以这样做：

   ~~~shell
   $ read packets _ _ time _ <<< "$info"
   ~~~

   这里`<<<` 是一个here-string，它让你直接把字符传递给命令的标准输入。

   （关于here-string，可以参数：[Here Strings](https://linux.die.net/abs-guide/x15683.html)）

7. 获取文件大小，并赋给变量

   ~~~shell
   $ size=$(wc -c < file)
   ~~~

   这行命令使用了命令替换操作符`$(...)`，这里我在第3点中有介绍过。它在`...`中运行命令，然后返回它的输出。在这里，这个命令是`wc -c < file`，它打印出了文件的字节数，然后输出被赋给了size变量。

8. 从路径中导出文件名

   假设你有一个文件`/path/to/file.ext`，你只想导出它的文件名`file.ext`，你要怎么做？有一个好的方案是使用参数展开功能：

   ~~~shell
   $ filename=${path##*/}
   ~~~

   这个命令使用了`${var##pattern}`，这个展开尝试在`$var`变量的开头匹配`pattern`。如果可以匹配上，这个展开的值就是从`$var`变量中将最长匹配的`pattern`删除的结果。

9. 从路径中导出目录名

   这里类似于前面的命令。假设你有一个文件`/path/to/file.ext`，你只想导出这个文件的目录。你可以再次使用参数展开：

   ~~~shell
   $ dirname=${path%/*}
   ~~~

   这次是使用了`${var%pattern}`参数展开，它会尝试在结尾用`pattern`匹配变量`$var`。如果可以匹配上，这个展开的值就是从`$var`的值中将最短匹配的`pattern`删除的结果。

   在这种情况下这个模式是`/*`，它会匹配`/path/to/file.ext`的结尾（它匹配了`/file.ext`）。然后这个结果就是目录`/path/to`，也就是删除了匹配的部分。

10. 快速对文件做复制

    假设你想复制文件`/path/to/file`到`/path/to/file_copy`。正常情况下你会这样操作：

    ~~~shell
    $ cp /path/to/file /path/to/file_copy
    ~~~

    但是你可以使用花括号扩展`{...}`以便更快的完成操作：

    ~~~shell
    $ cp /path/to/file{,_copy}
    ~~~

    花括号扩展是一种可以生成任意字符串的功能。这个特殊的例子中，`/path/to/file{,_copy}`生成了字符串`/path/to/file /path/to/file_copy`，完整的命令为`cp /path/to/file /path/to/file_copy`。

    同样的你可以快速的移走一个文件：

    ~~~shell
    $ mv /path/to/file{,_old}
    ~~~

    这个命令会扩展成`mv /path/to/file /path/to/file_old`。

    (关于花括号扩展，可以参考：[Brace expansion](https://wiki.bash-hackers.org/syntax/expansion/brace)









