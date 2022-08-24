### c++ 崩溃分析

### 1. coredump：捕获crash

#### coredump查看和开启

查看当前系统是否支持coredump

```sh
$ ulimit -c
0
```

如果为0，则表示coredump被关闭。

可以通过下面方式开启

```sh
$ ulimit -c unlimited
```

#### coredump文件位置

coredump文件默认存储位置与可执行文件在同一目录下，文件名为core。

可以通过/proc/sys/kernel/core_pattern来设置coredump文件生成的位置

#### coredump原理

参考:https://www.cnblogs.com/arnoldlu/p/11160510.html

### 2. gdb调试coredump

gdb ./a.out core

bt



### 3. gdb 常见命令

```shell
set args
breakpoint:b 打断点
continue:c	继续，类似f9
run	:r	运行程序
list:l	显示源码，一次显示10行，可以 l rownnumber，打印给定行号的上下行
next:n 	下一行step over,遇到函数调用，不进入函数内部，跳过函数调用
step:s	下一行step into,遇到函数调用，进入函数内部
print:p 打印变量
set var name=value	:设置变量的值
quit:q 	退出gdb环境
finish:用于跳出当前函数（一般step into之后，会finish跳出所在函数）
```



