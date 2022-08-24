# vim8+vimplus插件安装和配置

以ubuntu-server为例

# 1.安装vim8.2

## 1.1 查看vim版本

ubuntu-server默认自带vim7.4，可以通过vim -version查看

```shell
zhoukang@ubuntu:~/download/vim-master$ vim -version
VIM - Vi IMproved 7.4 (2013 Aug 10, compiled Oct 13 2020 16:04:38)
Garbage after option argument: "-version"
More info with: "vim -h"
```

## 1.2 下载vim8源码并解压

下载源码后解压，并进入源码目录

```shell
zhoukang@ubuntu:~/download/vim-master$ wget https://github.com/vim/vim/archive/master.zip
zhoukang@ubuntu:~/download/vim-master$ unzip vim-master.zip
zhoukang@ubuntu:~/download/vim-master$ ls
master.zip  vim-master
zhoukang@ubuntu:~/download/vim-master$ cd vim-master
```

## 1.3 运行vim配置

```shell
zhoukang@ubuntu:~/download/vim-master$ ls
ci  configure  CONTRIBUTING.md  Filelist  LICENSE  Makefile  nsis  pixmaps  READMEdir  README.md  README.txt  README_VIM9.md  runtime  src  tools  uninstall.txt  vimtutor.bat  vimtutor.com
```

先查看configure的选项参数

```shell
zhoukang@ubuntu:~/download/vim-master$ ./configure --help
```

参数很多，根据需要自行设置，一般设置--with-features为huge，开启最大特性，设置  --with-python3-config-dir，开启python3的配置。
  --with-features=TYPE    tiny, small, normal, big or huge (default: huge)
  --with-python3-config-dir=PATH  Python's config directory (deprecated)

```shell
zhoukang@ubuntu:~/download/vim-master$ ./configure --with-features=huge --with-python3-config-dir=/usr/lib/python3.5/config-3.5m-x86_64-linux-gnu
```

## 1.4 编译vim

```shell
zhoukang@ubuntu:~/download/vim-master$ make
```

## 1.5 安装vim

```shell
zhoukang@ubuntu:~/download/vim-master$ sudo make install
```

vim默认安装到/usr/local/bin/目录下

## 1.6 添加vim的环境变量

打开/etc/profile配置文件

```shell
zhoukang@ubuntu:~/download/vim-master$ sudo vim /etc/profile
```

在文件的最后一行添加如下内容

```
export PATH=/usr/local/bin:$PATH
```

使/etc/profile配置文件生效

```shell
zhoukang@ubuntu:~/download/vim-master$ source /etc/profile
```

## 1.7 再次验证vim版本

```shell
zhoukang@ubuntu:~/download/vim-master$ vim -version
VIM - Vi IMproved 8.2 (2019 Dec 12, compiled Jan  7 2022 17:08:33)
Garbage after option argument: "-version"
More info with: "vim -h"
```

# 2.安装vimplus插件

## 2.1 下载vimplus源码

```shell
zhoukang@ubuntu:~/download/vim-master$ git clone https://github.com/chxuan/vimplus.git ~/.vimplus
Cloning into 'vimplus'...
remote: Enumerating objects: 2350, done.
remote: Counting objects: 100% (43/43), done.
remote: Compressing objects: 100% (22/22), done.
remote: Total 2350 (delta 23), reused 38 (delta 21), pack-reused 2307
Receiving objects: 100% (2350/2350), 12.78 MiB | 2.53 MiB/s, done.
Resolving deltas: 100% (1452/1452), done.
Checking connectivity... done.
zhoukang@ubuntu:~/download/vim-master$ cd ~/.vimplus;ls
autoload  colors  fonts  ftplugin  help.md  install.sh  install_to_user.sh  LICENSE  README.md  screenshots  uninstall.sh  update.sh
zhoukang@ubuntu:~/.vimplus$ 
```

## 2.2 安装vimplus

```
zhoukang@ubuntu:~/.vimplus$ ./install.sh
```

# 3. vimplus使用

1.语法

2.帮助文档

3.乱码

# 4.vim语法

语法:a或者ar或者art

a:action

ar:action + range

art:action + range + text



action:动作

range:动作的范围

text:动作的内容

## action动作

y：复制yank

d：剪切delete

p：粘贴

c：修改

v：选中visual

x:删除字符

...

## Range范围

2  :  数量

i	: in， 里面

a	:all，里面+边界

t	:till，直到首个，不包括

f	：find，找到首个，包括



0是行首，$是行尾，^是首个非空字符，gg是文件开头，G是文件结尾

## text内容

{

[

(

,

a-z

w：word             

## vim 快捷键举例		

### 复制、剪切、粘贴

在vim中，复制(yank)、剪切(delete)、粘贴(put)，而不是ctrl+c和ctrl+v，因为c被占用了



y：yank复制

​		yy：复制一行

​		yiw:复制一个单词

​		2yy:复制2行



​		**yw**:复制单词后部分

​		**yb**:复制单词前部分

​		

d:	delete（这里理解成剪切的意思）

​		dd：剪切一行

​		diw:剪切一个单词("return",   光标在单词的任意一个位置，diw都是删除单词return)

​		2dd:剪切2行

​		d0:删除至行首

​		d$:删除至行尾

​		dgg:删除至文件头

​		dG:删除至文件尾



​		**db**：删除单词前部分

​		**dw**: 删除单词后部分

x:	剪切字符

​		x：删除一个字符

​		2x：删除2个字符

​		xx:删除两个字符（xx不支持删除行，因为删除单位是一个字符）

p： 粘贴

## 修改

c: 	change 修改

​		cc：修改一行（会先删除，再进入编辑状态）

​		ciw:修改一个单词

​		2cc:修改2行

​	

​		**cw**:修改单词后部分

​		**cb**: 修改单词前部分

​		

## 撤销

u

ctrl+r  重做最后的操作



## 复制粘贴系统剪切板

"+y

复制

"+d

剪切

"+p

粘贴

## 全文操作

ggvG

全文选择

ggdG

全文剪切

ggyG

全文复制



## 移动跳转

g: 	goto跳转的意思

​		gg:跳转到第一行

​		3gg:跳转到第3行

​		G:跳转到最后一行



移动跳转到某个字符

fb : 快速跳转到下一个b字符所在位置

​		之后按下f，继续跳转到下一个b字符位置

Fb：快速跳转到上一个b字符所在位置

​		之后按下F，继续跳转到上一个b字符位置



按照单词跳转

w:跳转到下一个单词

b：跳转到上一个单词





## **搜索内容**

/text：搜索内容为"text"的内容

\* 	: 搜索当前光标所在的单词内容

\#	: 搜索当前光标所在单词内容，同*作用相同



## 快速编辑

i：进入当前位置编辑

a：进入当前位置的后一个字符编辑

A：在行尾进行编辑



## 插入模式下粘贴外部内容到编辑器中

直接ctrl+v



一般会设置代码自动缩进set autoindent；但是设置之后，如果ctrl+v粘贴外部代码到当前位置时，缩进不混乱，

需要set paste，设置之后，在ctrl+v，然后set nopaste。

## 命令

### :set设置

:set xxx

### :s替换

:s/p1/p2/g

**:s/p1/p2/gc**

注意，s是删除一个字符并进入插入模式，:s是命令

:s中s的缩写是substitute（代替者）

**gc是全局替换之前询问是否替换，很好用**

### 范围操作

:100,200y

复制第100到200行

:100,200d

剪切100到200行



## vim寄存器

### 寄存器种类

vim操作的是寄存器，而不是系统剪切板。

在vim的命令模式下输入:reg，可以查看vim寄存器当前的内容

Type是寄存器的类型，按照寄存器类型，分为三种：l是行寄存器，b是块寄存器，c是字符寄存器

Name是寄存器的名字。

#### 		**无名寄存器**

​			"寄存器存放最近一次d/y/c/x/s操作的内容，

#### 		**数字寄存器**

​			0寄存器，是y寄存器，存放最近一次y操作的内容

​			1~9是d/c寄存器，存放最近一次d/c操作的内容，1~9像堆栈，第一次d/c操作，内容放入1寄存器，接着第二次d/c操作，1寄存器					中内容放入2寄存器，1寄存器存放最新的d/c操作的内容



#### 		**搜索寄存器**

​			/寄存器是搜索寄存器，存放最近一次/和*和#操作的内容



​		**系统剪切板寄存器**

​			*寄存器和+寄存器是系统剪切板寄存器，存放最近一次系统剪切的内容



#### 		**只读寄存器**

​			%寄存器是只读寄存器，存放的是当前文件名

​			:寄存器是命令寄存器，存放最近一次操作的命令，比如:set number 或者:reg "，或者:%s/g1/g2/g等命令。



#### 		

​		-寄存器，是x寄存器，存放最近一次x操作的内容



Content是内容





```
:reg
Type Name Content
  l  ""       bool    remove;^J
  l  "0       bool    remove;^J
  l  "1   public:^J
  l  "2   class ProtocolHandler;^J
  l  "3                                                                   ————————————————^J
  l  "4                                                                       版权声明：^J
  l  "5   }^J
  l  "6                                                                       825299^J
  l  "7   my name is zhoushiyi.ushiyi^Jmy name is zhoushiyi.ushiyi^Jmy name is zhoushiyhh.ushiyi^J
  l  "8   what's your name?zhous^J
  l  "9   8^J
  c  "r   eg^M^M；:reg^Mh"yy:reg^M^[<80><fd>a"yaw:^[<80><fd>a^[<80><fd>au:regh^M:reg^M
  c  "-   i
  c  ".
  c  ":   reg
  c  "%   text.txt
  c  "/   buffer
  c  "=   AutoPairsInsert('}')
```



### 寄存器的使用

#### 内容放入寄存器

 有些寄存器是默认寄存器，可以将操作内容，手动放到不同寄存器中，使用方法是

""yaw

"1yaw

"2yaw



""caw

"1caw



"1daw

#### 从寄存器中取出

""p

"1p

"2p



"+p

"=p



