---
title: Bash One-Liners Explained Part II Working with strings (译文)
tags:
- bash
categories:
- shell 
---

来自：[Bash One-Liners Explained, Part II: Working with strings](https://catonmat.net/bash-one-liners-explained-part-two)

# Part II 字符串处理

1. 生成从a到z的字母表

   ~~~shell
   $ echo {a..z}
   ~~~

   这行命令使用了花括号扩展。花括号扩展是一种可以生成任意字符串的功能。这一行命令使用了形如`{x..y}`的顺序表达式，这里x和y是单个字符。顺序表达式按照字符顺序展开从x到y的每个字符。

   如果运行的话，你可以得到a-z间的每个字符：

   ~~~shell
   $ echo {a..z}
   a b c d e f g h i j k l m n o p q r s t u v w x y z
   ~~~

2. 生成从a到z字符间没有空格的字母表

   ~~~shell
   $ printf "%c" {a..z}
   ~~~

   这是一个99.99%的bash用户都不知道的超棒的bash技巧。如果你给printf函数提供一个列表清单，它会循环应用这个列表知道列表为空！作为循环的printf！没有比这更棒的了。

   在这行命令中printf的格式是`%c`，意思是一个字符，参数是以空格区分的从a到z的所有字母。所以printf所做的工作就是它会迭代列表并依次输出字母，直到运行完所有的字母。

   如果运行的话，这是它的结果：

   ~~~shell
   abcdefghijklmnopqrstuvwxyz
   ~~~

   因为格式字符是`%c`并且不包括`\n`，所以它的输出结果中不包含换行符。如果需要以换行符结束的话，只需要把`$'\n'`添加到字符列表以输出：

   ~~~shell
   $ printf "%c" {a..z} $'\n'
   ~~~

   `$'\n'`是bash中惯用的一种用来表示换行符的方法。于是printf就只需要输出a到z的字符，然后是换行符。

   另外一种往结尾添加换行符的方式是打印printf的输出：

   ~~~shell
   $ echo $(printf "%c" {a..z})
   ~~~

   这行命令使用了运行`printf "%c" {a..z}`的命令替换，使用它的输出替换了命令。echo打印了输出并且添加了换行符。

   想要在一列中输出所有字母？在每个字符后面都添加一个换行符！

   ~~~shell
   $ printf "%c\n" {a..z}
   ~~~

   输出如下：

   ~~~shell
   a
   b
   ...
   z
   ~~~

   想要快速的把printf的输出赋给一个变量？使用-v参数：

   ~~~shell
   $ printf -v alphabet "%c" {a..z}
   ~~~

   这会把abcdefghijklmnopqrstuvwxyz赋给变量$alphabet。

   同样的可以生成一个数字列表，比如从1到100：

   ~~~shell
   $ echo {1..100}
   ~~~

   输出如下：

   ~~~shell
   1 2 3 ... 100
   ~~~

   另外，如果你忘记了这种方式，亦可以使用外部seq工具来生成一个数字序列：

   ~~~shell
   $ seq 1 100
   ~~~

3. 生成有0为前导的0到9

   ~~~shell
   $ printf "%02d " {0..9}
   ~~~

   我们这里再次使用了printf的循环功能。这一次格式为`%02d`，意思是使用0填充整数到2个位置。要循环遍历的项目是由花括号扩展生成的数字0到9（在之前的命令中有介绍过）。

   输出：

   ~~~shell
   00 01 02 03 04 05 06 07 08 09
   ~~~

   如果你的bash版本为4，你可以使用花括号扩展的新功能来达到同样的目的：

   ~~~shell
   $ echo {00..09}
   ~~~

   更早版本的bash没有这个功能。

4. 创建30个英文单词

   ~~~shell
   $ echo {w,t,}h{e{n{,ce{,forth}},re{,in,fore,with{,al}}},ither,at}
   ~~~

   这是一个对花括号展开的滥用。看看它创建了什么：

   ~~~shell
   when whence whenceforth where wherein wherefore wherewith wherewithal whither what then thence thenceforth there therein therefore therewith therewithal thither that hen hence henceforth here herein herefore herewith herewithal hither hat
   ~~~

   非常厉害！

   这是它运行的原因-你可以使用花括号扩展创建单词或者符号列表。举例如下，如果你这样做：

   ~~~shell
   $ echo {a,b,c}{1,2,3}
   ~~~

   它产生的结果为`a1 a2 a3 b1 b2 b3 c1 c2 c3`，它采用第一个a，并将它和`{1,2,3}`连接在一起生成了`a1 a2 a3`。然后它使用第二个b，将它和`{1,2,3}`连接在一起，然后再对c做同样的操作。

   所以这行命令是花括号的智能连接，当使用扩展时就可以生成所有这些英文单词。

5. 创建相同字符串的10个副本

   ~~~shell
   $ echo foo{,,,,,,,,,,}
   ~~~

   这行命令同样使用了花括号扩展。这里是把foo和10个空字符连接在了一起，所以输出是foo的10个副本：

   ~~~shell
   foo foo foo foo foo foo foo foo foo foo foo
   ~~~

6. 连接两个字符串

   ~~~shell
   $ echo "$x$y"
   ~~~

   这行命令简单的把两个变量连接在了一起。如果变量x包含foo，同时y包含bar，结果则为foobar。

   注意到`$x$y`被引号引用，如果我们不引用，echo命令会把$x$y解释成常规参数，它将会首先解析它们以确定是否包含命令行开关。所以如果$x包含以-开头的内容，对echo来讲，它就是一个命令行参数而不是一个echo的参数：

   ~~~shell
   x=-n
   y=" foo"
   echo $x$y
   ~~~

   输出：

   ~~~shell
   foo
   ~~~

   和正确的方式比较：

   ~~~shell
   x=-n
   y=" foo"
   echo "$x$y"
   ~~~

   输出：

   ~~~shell
   -n foo
   ~~~

   如果你需要把两个连接的字符赋给一个变量，你可以忽略引号：

   ~~~shell
   var=$x$y
   ~~~

7. 将给定的字符分割成字符串

   假设变量$str中有一个字符串foo-bar-baz，你想要使用破折号来分割并迭代它。你可以简单的见read和IFS连接起来使用：

   ~~~shell
   $ IFS=- read -r x y z <<< "$str"
   ~~~

   这里我们使用read x命令来从输入读取数据，然后将数据赋给变量x y z。我们把IFS设置成-，因为这个变量被拿来作为分隔符。如果多个变量名称被指定给read命令，IFS会被用来分割输入的行，所以每个变量都可以得到一个输入的列。

   在这个命令中$x获取了foo，$y获取了bar，$z或者了baz。

   另外还需要注意到`<<<`操作符。这个一个here-string操作符，它允许传输字符串到命令的输入端。在这个例子中字符串$str被传递给read命令的输入端。

   你也可以将被分割的列放到数组中：

   ~~~shell
   $ IFS=- read -ra parts <<< "foo-bar-baz"
   ~~~

   read的-a参数使得它会将分割的单词放到数组中。在这个例子中，数组是parts，你可以通过`${parts[0]}`，`${parts[1]}`和`${parts[2]}`来访问数组元素。或者是通过`${parts[@]}`访问所有的值。

8. 逐个字符处理字符串

   ~~~shell
   $ while IFS= read -rn1 c; do
       # do something with $c
   done <<< "$str"
   ~~~

   这里我们给read命令使用-n1的参数，使它一次读取一个字符，类似的我们可以使用-n2让它一次读取2个字符。

9. 在字符串中使用bar代替foo

   ~~~shell
   $ echo ${str/foo/bar}
   ~~~

   这行命令使用形如`${var/find/replace}`的参数扩展。它会在变量字符var中寻找到find，然后使用replace替代。真的很简单！

   (注意，这里只会替换第一个找到的值)

   将所有的foo都用bar替换，使用`${var//find/replace}`的格式：

   ~~~shell
   $ echo ${str//foo/bar}
   ~~~

10. 检查字符串是否匹配某种模式

    ~~~shell
    $ if [[ $file = *.zip ]]; then
        # do something
    fi
    ~~~

    这行命令是在$file匹配*.zip的情况下做一些操作。这个是一个简单的通配符匹配，可以用是符号`*?[...]`来匹配。\*号匹配任意字符，?匹配单字符，[...]匹配任意...代表的字符或者一个字符集。

    这里是另外一个匹配Y或者y的例子：

    ~~~shell
    $ if [[ $answer = [Yy]* ]]; then
        # do something
    fi
    ~~~

11. 检查字符串是否匹配一个正则表达式

    ~~~shell
    $ if [[ $str =~ [0-9]+\.[0-9]+ ]]; then
        # do something
    fi
    ~~~

    这行命令测试了$str是否匹配正则`[0-9]+.[0-9]+`，它代表的是两个用点连接的数字。正则表达式在`man 3 regex`中有描述。

12. 计算字符串的长度

    ![](https://images-pigo.oss-cn-beijing.aliyuncs.com/20220703232140.png)

    我们使用了参数表达式![](https://images-pigo.oss-cn-beijing.aliyuncs.com/20220703232241.png),它会返回在变量str中的字符长度。很简单。

    (这里不能输入![](https://images-pigo.oss-cn-beijing.aliyuncs.com/20220703232241.png)字符，否则显示不正常，怀疑是Typora的bug。)

13. 从字符中提取子字符串

    ~~~shell
    $ str="hello world"
    $ echo ${str:6}
    ~~~

    这行命令从hello world中提取了world。它使用了子字符串扩展。通常子字符串扩展看上去像`${var:offset:length}`，它会在var中从索引offset出开始提取length长度的字符。我们这里的命令忽略了length，这是的它会提取从offset 6开始往后的所有字符。

    这里有另外一个例子：

    ~~~shell
    $ echo ${str:7:2}
    ~~~

    输出：

    ~~~shell
    or
    ~~~

14. 将字符串转成大写

    ~~~shell
    $ declare -u var
    $ var="foo bar"
    ~~~

    在bash中declare命令声明变量以及它的属性。在这个例子中，我们给变量var一个-u的属性，当它被赋值时，它的内容会被转变成大写字母。现在你显示它，就会发现它的内容都是大写的：

    ~~~shell
    $ echo $var
    FOO BAR
    ~~~

    注意-u参数是在bash 4时被引用的。同样你可以使用bash 4的另一种可以使var变量中字符串变成大写的方法，就是`${var^^}`参数扩展：

    ~~~shell
    $ str="zoo raw"
    $ echo ${str^^}
    ~~~

    输出：

    ~~~shell
    ZOO RAW
    ~~~

15. 将字符串转变为小写

    ~~~shell
    $ declare -l var
    $ var="FOO BAR"
    ~~~

    与前一个命令类似，declare的-l参数给var变量设置了一个小写的属性，这就使得它一直都是小写的：

    ~~~shell
    $ echo $var
    foo bar
    ~~~

    -l参数也只存在于bash 4以及后面的版本中。

    另外一种使得字符串小写的方式是使用`${var,,}`参数扩展：

    ~~~shell
    $ str="ZOO RAW"
    $ echo ${str,,}
    ~~~

    输出：

    ~~~shell
    zoo raw
    ~~~

    