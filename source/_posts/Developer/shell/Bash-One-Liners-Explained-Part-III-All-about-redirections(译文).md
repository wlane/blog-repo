---
title: Bash One-Liners Explained Part III All about redirections(译文)
tags:
- bash
categories:
- shell
---

来自：[Bash One-Liners Explained, Part III: All about redirections (catonmat.net)](https://catonmat.net/bash-one-liners-explained-part-three)

# Part III 关于重定向的一切

一旦意识到重定向是有关于操作文件描述符的，你就会知道使用重定向是很简单的。当bash启动的时候，它会打开三个文件描述符：标准输入（文件描述符0），标准输出（文件描述符1），标准错误（文件描述符2）。你也可以打开和关闭更多的文件描述符。你可以复制文件描述符，也可往其中写入内容或者从中读取内容。

文件描述符总是指定一些文件（除非被关闭了）。通常情况下，当bash启动时，三个文件描述符都是指向你的终端。读取你在终端中输入的内容以及把输出发送给终端。

假设你的终端是`/dev/tty0`，下面是bash启动时文件描述符表的样子：

![](https://images-pigo.oss-cn-beijing.aliyuncs.com/20220705214409.png)

当bash启动一个命令时，它会启动一个子进程（查看`man 2 fork`），这个子进程会继承所有父进程的文件描述符，然后设置重定向，最后执行命令(查看`man 3 exec`)。

想要成为一个bash重定向的专家，你需要可视化当重定向发生时，文件描述符是怎么变化的。图形化会帮忙你理解这一切。

1. 重定向标准输出到文件

   ~~~shell
   $ command >file
   ~~~

   操作符`>`是输出重定向操作符。Bash首先尝试打开一个文件，如果成功的话，它会将命令的输出写入到刚打开的文件。如果打开文件失败，则整个命令失败。

   命令`command > file`和`command 1> file`是一样的。数字1代表的是输出，它是标准输出的描述符号。

   下面是文件描述符表的变化。Bash打开file，使用指向文件file的文件描述符替换文件描述符1。所以所有写入到文件描述符1的输出都会写入到file中：

   ![](https://images-pigo.oss-cn-beijing.aliyuncs.com/20220705220156.png)

   通常你可以写成`command n> file`，这会重定向文件描述符n到file。例如：

   ~~~shell
   $ ls > file_list
   ~~~

   重定向ls命令的结果到file_list文件中。

2. 重定向标准错误到文件

   ~~~shell
   $ command 2> file
   ~~~

   这里Bash会重定向标准错误到文件，2代表标准错误。

   这里是文件描述符表的变化：

   ![](https://images-pigo.oss-cn-beijing.aliyuncs.com/20220705222209.png)

   Bash打开文件file，获取这个文件的文件描述符，使用这个文件的文件描述符替换文件描述符2。现在任何写到标准错误的内容都写到了文件中。

3. 同时重定向标准输出和标准错误到文件

   ~~~shell
   $ command &>file
   ~~~

   这行命令使用`&>`操作符同时重定向标准输出和错误到文件。这是bash的快速重定向两个流到文件的快捷方式。

   下面是在bash重定向两个流之后文件描述符表的样子：

   ![](https://images-pigo.oss-cn-beijing.aliyuncs.com/20220706211644.png)

   你可以看到此时标准输出和标准错误都指向了file。所以任何写到标准输出到错误的内容到写到了文件。

   有好几种方式可以重定向两个流到相同的位置。你可以依次重定向两个流：

   ~~~shell
   $ command >file 2>&1
   ~~~

   这是一种更常用的重定向两个流到文件的方式。首先标准输出重定向到文件file，然后将标准错误复制为和标准输出一样的。所以两个流最终都指向了文件file。

   当bash遇到多个重定向时，它按照从左到右的顺序处理它们。让我们浏览一下整个步骤看看这些都是如何发生的。bash在运行任何命令之前，文件描述符表看上去是这样的：

   ![](https://images-pigo.oss-cn-beijing.aliyuncs.com/20220705214409.png)

   现在bash开始处理第一个重定向`>file`。我们之前已经见过这种操作，它将输出指向文件file：

   ![](https://images-pigo.oss-cn-beijing.aliyuncs.com/20220705220156.png)

   接下来bash看到第二个重定向`2>&1`。我们之前没有见过这种重定向。它会把文件描述符2复制成文件描述符1的副本，这样我们就得到：

   ![](https://images-pigo.oss-cn-beijing.aliyuncs.com/20220706213122.png)

   两个流都重定向到了文件file。

   但是这里要小心，这种写法：

   ~~~shell
   $ command >file 2>&1
   ~~~

   与这种写法不同：

   ~~~shell
   $ command 2>&1 >file
   ~~~

   在bash中重定向的顺序是有意义的。上面这种命令只是重定向标准输出到文件。标准错误还是指向终端。要理解这是如何发生的，让我们再次浏览这些步骤。一开始在运行命令之前文件描述符表是这样的：

   ![](https://images-pigo.oss-cn-beijing.aliyuncs.com/20220705214409.png)

   现在bash从左到右处理重定向。它先看到`2>&1`，它复制标准错误到标准输出，文件描述符表变成：

   ![](https://images-pigo.oss-cn-beijing.aliyuncs.com/20220706213802.png)

   接下来bash看到第二个重定向，它重定向标准输出到文件file：

   ![](https://images-pigo.oss-cn-beijing.aliyuncs.com/20220706213857.png)

   你看到这里发生什么了吗？现在标准输出指向到了文件file，但是标准错误仍然指向的是终端。任何写到标准错误仍然会打印到终端上。所以需要非常小心重定向的顺序。

   还需要注意到在bash中，这样写：

   ~~~shell
   $ command &>file
   ~~~

   和这样的命令完全相同：

   ~~~shell
   $ command >&file
   ~~~

   更推荐第一种形式的写法。

4. 忽略命令的标准输出

   ~~~shell
   $ command > /dev/null
   ~~~

   特殊文件/dev/null会忽略所有写入它的数据。所以我们这里做的是重定向标准输出到这个文件，然后这些输出都被丢弃。从文件描述符表的角度来看它是什么样的：

   ![](https://images-pigo.oss-cn-beijing.aliyuncs.com/20220706214656.png)

   同样的，结合上一个命令，我们可以同时丢弃标准输出和标准错误：

   ~~~shell
   $ command >/dev/null 2>&1
   ~~~

   或者简单的：

   ~~~shell
   $ command &>/dev/null
   ~~~

   这种行为的文件描述符表就像这样：

   ![](https://images-pigo.oss-cn-beijing.aliyuncs.com/20220706214939.png)

5. 重定向文件的内容到一个命令的输入

   ~~~shell
   $ command <file
   ~~~

   这里bash在执行命令之前此时打开一个文件来读。如果打开失败，bash带着错误退出，不再运行任何命令。如果打开成功，bash使用打开文件的文件描述符作为命令的输入文件描述符。

   完成上述流程后，文件描述符表如下：

   ![](https://images-pigo.oss-cn-beijing.aliyuncs.com/20220706215831.png)

   这里有一个例子，当你要读取文件的第一行到变量中，你可以简单的这样处理：

   ~~~shell
   $ read -r line < file
   ~~~

   bash内置命令read从标准输入读取单行。通过使用这个输入重定向操作符`<`，我们就可以将其设置成中文件中读取行。

6. 重定向一堆文本到一个命令的输入

   ~~~shell
   $ command <<EOL
   your
   multi-line
   text
   goes
   here
   EOL
   ~~~

   