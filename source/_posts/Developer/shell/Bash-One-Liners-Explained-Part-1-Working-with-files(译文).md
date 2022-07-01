---
title: Bash One-Liners Explained Part 1 Working with files(译文)
tags:
- bash
categories:
- shell
---

来自：[Bash One-Liners Explained, Part I: Working with files (catonmat.net)](https://catonmat.net/bash-one-liners-explained-part-one)

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

   这行命令使用了一个不同的输出重定向字符“>>”，它会追加内容到文件。如果文件不存在则创建。新加的字符串后面会跟着换行符，如果你不想在字符串后面存在换行符，给echo命令加上-n参数：

   ~~~shell
   $ echo -n "foo bar baz" >> file
   ~~~

3. 读取文件首行并且把它赋给变量

   ~~~shell
   $ read -r line < file
   ~~~

   这行指令使用了内置的bash命令read和输入重定向操作符“<”。read命令读取标准输入的一行并把它赋给变量line。-r参数表示输入的都是原始数据，也就是说反斜杠不会被忽略（它们会被原样读取），这个“<”重定向的命令打开文件准备读取，使得它成为read命令的标准输入。

   read命令会移除所有出现在特殊变量 IFS 中的字符，IFS代表了内部字段分隔符，它用来字符扩展后的分隔以及使用read内置命令把行分隔成单词。默认情况下，IFS包含空格、制表符和换行符，也就是说开头和结尾的空格和制表符都会被删除。如果你想保留它们，就需要把IFS设置为空值：

   ~~~shell
   $ IFS= read -r line < file
   ~~~

   这将针对这条命令修改IFS的值，并且保证line变量读取的第一行是包含开头和结尾空格的原始数据。

   另外一种从文件中读取第一行到变量中的方式是：

   ~~~shell
   $ line=$(head -1 file)
   ~~~

   这行命令使用了命令替换操作符“$(...)”，它在“...”部分运行命令，并返回输出。这里的情况下，命令是输出第一行内容的“head -1 file”。然后将输出赋值给变量line。使用“$(...)”等价“\`...\`”，所以你可以写成：

   ~~~shell
   $ line=`head -1 file`
   ~~~

   不过在bash中，使用$()是更好的方式，因为它更简洁和易于嵌套。

4. 依次读入文件的每一行

   