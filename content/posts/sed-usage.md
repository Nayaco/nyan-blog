---
title: "Sed Usage"
date: 2021-03-22T00:51:34+08:00
draft: false
tags: ["Linux Command"]
summary: "sed"
---

`sed`是一个流编辑器(Stream editor), 流编辑器用于对输入流（文件或来自管道的输入）执行基本的文本转换. 尽管在某种程度上类似于允许脚本化编辑的编辑器(例如ed). 但`sed`仅对输入进行一次传递即可工作, 因此效率更高. 但是, 它具有sed在管道中过滤文本的能力, 这使其与其他类型的编辑器特别有区别. 

### Basic Syntax & Workflow

sed命令有两种基本写法:

``` shell
$ sed [-n] [-e] 'command(s)' files 
$ sed [-n] -f scriptfile files
```
`-n`的意思和`--quiet`是一样的, 就显示处理后的结果. 一般脚本中需要流编辑的时候就会加上这条参数.

`-e`就是`--expression`后面接表达式, 字面理解就可以.

表达式的常用动作:
+ a: append
+ c: replace(和s的取代不同, 不是接表达式, 是按行取代)
+ d: delete
+ i: insert
+ p: print
+ s: replace 

Sed命令的workflow如下:
```
read 1 line -> excute line -> display line
```
说白了就是按行执行的方式.

本文中使用的样例输入如下:
```
1) A Storm of Swords, George R. R. Martin, 1216 
2) The Two Towers, J. R. R. Tolkien, 352 
3) The Alchemist, Paulo Coelho, 197 
4) The Fellowship of the Ring, J. R. R. Tolkien, 432 
5) The Pilgrimage, Paulo Coelho, 288 
6) A Game of Thrones, George R. R. Martin, 864
```

### Commands

+ **Delete Command**

删除命令
```
[address1[,address2]|expr]d
```
删除从address1到address2之间的内容, 如果不指定address2, 则仅删除address1的内容.

``` shell
$ sed '4d' book.txt
1) A Storm of Swords, George R. R. Martin, 1216 
2) The Two Towers, J. R. R. Tolkien, 352 
3) The Alchemist, Paulo Coelho, 197 
5) The Pilgrimage, Paulo Coelho, 288 
6) A Game of Thrones, George R. R. Martin, 864
$ sed '2,4d' book.txt 
1) A Storm of Swords, George R. R. Martin, 1216 
5) The Pilgrimage, Paulo Coelho, 288 
6) A Game of Thrones, George R. R. Martin, 864
$ sed 'd' book.txt
```
特别, 上面的第三条命令表示全部删除.

+ **Read Command**

```
[address]r file
```
"我们可以指示SED读取文件的内容并在特定条件匹配时显示它们."
虽然感觉是有点不明所以的命令, 但是大概哪里有用吧.

+ **Write Command**

写入命令
```
[address1[,address2]|expr]w file
```
将address1和address2之间的内容写入file中.
``` shell
$ sed -n 'w book.bak' book.txt 
$ sed -n '2~2w book' book.txt
$ sed -n -e '/Martin/ w Martin.txt' -e '/Paulo/ w Paulo.txt' book.txt
$ sed -n '/Th*/w book.bak' book.txt
```
这些命令可以自己写一写, 这里就不写出来了, 应该可以看懂我写的吧.

+ **Append Command**

在后面添加
```
[address|expr]a Text
```
在address后面添加Text内容.
``` shell
$ sed '$a\ \ 7) NewlineXD' book.txt
1) A Storm of Swords, George R. R. Martin, 1216
...
6) A Game of Thrones, George R. R. Martin, 864
  7) NewlineXD
```
`$`字符表示最后一行, 第一行的话就用数字1表示就可以了.
+ Insert Command
Insert命令和Append差不多, 不解释了跳过.

+ **Change Command**

按行替换
```
[address1[,address2]|expr]c Replace Text
```
把address改为相应内容.
``` shell
$ sed '/\(The\|Storm\)/c Balabala' book.txt
Balabala
...
Balabala
6) A Game of Thrones, George R. R. Martin, 864
```
可以使用正则去匹配行, 匹配结果按行表示.

+ **Translate Command**

翻译
```
[address1[,address2]]y/list-1/list-2/
```
翻译命令必须要求list-1和list-2长度相同, 同时不支持正则表达式和字符类型.
```
$ echo "1 5 15 20" | sed 'y/151520/IVXVXX/' 
I V IV XX 
```

+ **L Command**

L命令用于转义文本中的不可见字符, 比如\t, $, 当然也是按行执行的.
```
[address1 [，address2]] l 
[address1 [，address2]] l [len]
```
下面那种写法Len表示输出多少字符后换行(wrapping).
``` shell
$ sed -n 's/ /\t/g;l 30' book.txt
1)\tA\tStorm\tof\tSwords,\tGe\
orge\tR.\tR.\tMartin,\t1216\t$
2)\tThe\tTwo\tTowers,\tJ.\tR.\
\tR.\tTolkien,\t352\t$
3)\tThe\tAlchemist,\tPaulo\tC\
oelho,\t197\t$
4)\tThe\tFellowship\tof\tthe\
\tRing,\tJ.\tR.\tR.\tTolkien,\
\t432\t$
5)\tThe\tPilgrimage,\tPaulo\t\
Coelho,\t288\t$
6)\tA\tGame\tof\tThrones,\tGe\
orge\tR.\tR.\tMartin,\t864$
```
+ **Quit Command**

Quit
```
[address]q 
[address]q [value]
```
匹配到address就退出value是返回码, 没什么好说的跳过.

+ **Execute Command**

执行外部命令
```
[address1[,address2]]e [command]
```
比如我们想知道sed执行一行东西需要多少时间我们可以
``` shell
$ sed 'e date +%s.%N' book.txt
1617173673.715913570
1) A Storm of Swords, George R. R. Martin, 1216 
1617173673.722971126
2) The Two Towers, J. R. R. Tolkien, 352 
1617173673.729997560
3) The Alchemist, Paulo Coelho, 197 
1617173673.736437244
4) The Fellowship of the Ring, J. R. R. Tolkien, 432 
1617173673.742550613
5) The Pilgrimage, Paulo Coelho, 288 
1617173673.748491791
6) A Game of Thrones, George R. R. Martin, 864
```
这样就知道一行运行5ms, 当然也可以有其他的语法, 比如直接不加command, sed就会认为缓冲区的内容就是外部命令.
``` shell
$ echo 'cal' | sed 'e'
     March 2021     
Su Mo Tu We Th Fr Sa
    1  2  3  4  5  6
 7  8  9 10 11 12 13
14 15 16 17 18 19 20
21 22 23 24 25 26 27
28 29 30 31 
```
上述命令和直接cal没什么区别,除了不能输出shell的颜色以外, 不过pipe也不会吧颜色这些字符传过去就是了.

+ **Miscellaneous Commands**

N命令用来表示Next, n也是next, 不过N不清空缓冲区罢了, 而是将下一行直接接到当前行上
``` shell
$ seq 10 | sed -n '/3/{n;p}'
4
$ seq 10 | sed -n '/3/{N;p}'
3
4
```

还有像v命令用来匹配当前sed版本, 感觉没什么用的命令, 比如当前是sed-4.8:
``` shell
$ sed 'v 4.9' book.txt
sed: -e expression #1, char 5: expected newer version of sed
```

### Regular Expression

sed可以采用正则匹配, 而且正则匹配功能很强, 使用正则匹配功能, sed几乎可以实现大部分文本处理应用的功能, 比如head grep什么的.

+ **Start of Line(^)**

使用\^可以匹配行首

+ **End of Line($)**

使用\$可以匹配行末

上述两种正则匹配合起来可以匹配很多有用的东西, 比如说去掉空行:

``` shell
$ sed '/^$/d' file
```
+ **Single(.)**

+ **Zero or One Occurrence(\\?)**

+ **One or More Occurrence(\\+)**

+ **Zero or More Occurrence(*)**

+ **N to \[M\] Occurrence(\\{N\[,M\]})** *有一说一M可以是空的,此时表示无上限*

上述这些匹配规则都差不多都是用来匹配指定数目的字符.

+ **Pipe(|)**

pipe表示或的意思, 用法`\(1\|3\)`

+ **Set(\[\])** 
+ **Exclusive Set(\[^\])** 
+ **Character Range(\[-\])**

差不多的东西, 用法`[^a-z]`.

### Words at Last

本文是随感而发, 想到这个东西就随手写了一下, 大部分内容不是我自己写的(抄来的), 只是整理翻译了一下, 如果觉得看到过就跳过(以防有人阴阳怪气).  

**相关资料:**
+ Sed Tutorial: [tutorialspoint.com/sed/index.htm](https://tutorialspoint.com/sed/index.htm)
+ Regular Expr: [gnu.org/software/sed/manual/html_node/Regular-Expressions.html](https://gnu.org/software/sed/manual/html_node/Regular-Expressions.html)