---
layout: article
title: Input/Output and Command-Line Processing
excerpt_separator: <!--more-->
---
详细的解释shell怎样执行命令的。
<!--more-->

# I/O Redirectors
我们熟悉的从I/O重定向有<,>,|，虽然我们知道了这些可以完成差不多90%以上的工作，但是你应该知道bash不只是支持这些。
下面我尝试着完整的列举bash支持的I/O redirector。
```
cmd1 | cmd2
```
Pipe; take standard output of cmd1 as standard input to cmd2 .
管道，把cmd1的标准输出作为cmd2的标准输入
```
> file
```
Direct standard output to file .
标准输出到文件
```
< file
```
Take standard input from file .
从文件获取标准输入
```
>> file
```
Direct standard output to file ; append to file if it already exists.
标准输出到文件，如果文件存在则追加到文件结尾。
```
>| file
```
Force standard output to file even if noclobber is set.
```
n >| file
```
Force output to file from file descriptor n even if noclobber is set.
```
<> file
```
Use file as both standard input and standard output.
使用文件作为标准输入输出。
```
n <> file
```
Use file as both input and output for file descriptor n .
```
<< label
```
Here-document; see text.
```
n > file
```
Direct file descriptor n to file .
```
n < file
```
Take file descriptor n from file .
```
n >> file
```
Direct file descriptor n to file ; append to file if it already exists.
```
n >&
```
Duplicate standard output to file descriptor n .
```
n <&
```
Duplicate standard input from file descriptor n .
```
n >&m
```
File descriptor n is made to be a copy of the output file descriptor.
```
n <&m
```
File descriptor n is made to be a copy of the input file descriptor.
```
&>file
```
Directs standard output and standard error to file .
```
<&-
```
Close the standard input.
```
>&-
```
Close the standard output.
```
n >&-
```
Close the output from file descriptor n .
```
n <&-
```
Close the input from file descriptor n .
```
n>&word
```
If n is not specified, the standard output (file descriptor 1) is used. If the digits in word do not
specify a file descriptor open for output, a redirection error occurs. As a special case, if n is omitted,
and word does not expand to one or more digits, the standard output and standard error are
redirected as described previously.
```
n<&word
```
If word expands to one or more digits, the file descriptor denoted by n is made to be a copy of that
file descriptor. If the digits in word do not specify a file descriptor open for input, a redirection error
occurs. If word evaluates to -, file descriptor n is closed. If n is not specified, the standard input (file
descriptor 0) is used.
```
n>&digit-
```
Moves the file descriptor digit to file descriptor n , or the standard output (file descriptor 1) if n is
not specified.
```
n<&digit-
```
Moves the file descriptor digit to file descriptor n , or the standard input (file descriptor 0) if n is not
specified. digit is closed after being duplicated to n .

上述内容有一些有一些很有用，其他的主要用作系统编程。当然有些我也不是很清楚，姑且暂时记录下来，以便事后回顾。

## << label
Here document.将输入强制作为shell的标准输入，直到只包含label的一行停止。label中的输入就是here-document。如：
```
cmd << label
  Here Document Content
label
```
这里的label并不是固定的，可以是EOF，或者是其他的字符，但要成对出现。
here document比较适合在脚本中使用，可以批量的输入命令。比如登录ftp、redis，使用ed修改文件，甚至是发送邮件。

## redis
登录redis并获取某个key值。-a指定密码，get是redis的命令，用来获取key值，然后退出exit登录。
```
redis-cli -a password<<EOF
get key
exit
EOF
```

## ed
下面的脚本需要保存到脚本文件中执行。它接收一个文件参数，删除参数文件中第一行到第一个空行的部分。
1,/^[ ]*$/d，删除从第一行到第一个空行的内容；w把修改写入文件；q退出。
```
ed $1 << EOF
1,/^[ ]*$/d
w
q
EOF
```
