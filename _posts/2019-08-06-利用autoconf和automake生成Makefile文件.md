---
layout: post
title: 利用autoconf和automake生成Makefile文件
date: 2019-08-06 23:30:00
tags: linux编程
categories: 程序/代码
author: PythonPig
---
* content
{:toc}

在之前的项目中，编译较复杂开源项目或安装软件时经常使用./config、make、make install等命令，用的比较多比较熟了，但没有对其过程进行深入的学习，目前手里的项目需要用到相关的东西，就在这里简单的了解一下使用autoconf和automake生成makefile的过程。  
{:refdef: style="text-align: center;"}
![autoconfig](https://github.com/PythonPig/PythonPig.github.io/blob/master/images/利用autoconf和automake生成Makefile文件/autoconf.jpg?raw=true)
{: refdef}
图片来源于https://everycity.co.uk/alasdair/2011/03/autoconf-automake-and-libtoolized-version-of-bzip2/




今天这篇文章主要是转载IBM developerWorks的一篇文章（见参考部分第一篇文章），文章逻辑非常清楚，在转载的同时，对文章部分内容作了小的修改和增加，文章版权属于原作者。

### \#0x00 写在前面
无论是在Linux还是在Unix环境中，make都是一个非常重要的编译命令。不管是自己进行项目开发还是安装应用软件，我们都经常要用到make或 make install。利用make工具，我们可以将大型的开发项目分解成为多个更易于管理的模块，对于一个包括几百个源文件的应用程序，使用make和 makefile工具就可以轻而易举的理顺各个源文件之间纷繁复杂的相互关系。  

但是如果通过查阅make的帮助文档来手工编写Makefile,对任何程序员都是一场挑战。幸而有GNU 提供的Autoconf及Automake这两套工具使得编写makefile不再是一个难题。  

本文将介绍如何利用 GNU Autoconf 及 Automake 这两套工具来协助我们自动产生 Makefile文件，并且让开发出来的软件可以像大多数源码包那样，只需"./configure", "make","make install" 就可以把程序安装到系统中。  

### \#0x01 模拟需求
假设源文件按如下目录存放，如图1所示，运用autoconf和automake生成makefile文件。  
图1 文件目录结构  
{:refdef: style="text-align: center;"}
![目录结构](https://github.com/PythonPig/PythonPig.github.io/blob/master/images/利用autoconf和automake生成Makefile文件/目录结构.jpg?raw=true)
{: refdef}

假设src是我们源文件目录，include目录存放其他库的头文件，lib目录存放用到的库文件，然后开始按模块存放，每个模块都有一个对应的目录，模块下再分子模块，如apple、orange。每个子目录下又分core，include，shell三个目录，其中core和shell目录存放.c文件，include的存放.h文件，其他类似。  

样例程序功能：基于多线程的数据读写保护（联系原作者获取工程和源码（normalnotebook@126.com）或[点击这里下载源码](https://raw.githubusercontent.com/PythonPig/PythonPig.github.io/master/images/利用autoconf和automake生成Makefile文件/project.rar)）。  

### \#0x02 工具简介
所必须的软件：autoconf/automake/m4/perl/libtool（其中libtool非必须）。  

autoconf是一个用于生成可以自动地配置软件源码包，用以适应多种UNIX类系统的shell脚本工具，其中autoconf需要用到m4(主要处理宏相关内容)，便于生成脚本，m4文件有由aclocal命令生成。automake是一个从Makefile.am文件自动生成Makefile.in的工具。为了生成Makefile.in，automake还需用到perl，由于automake创建的发布完全遵循GNU标准，所以在创建中不需要perl。libtool是一款方便生成各种程序库的工具。  

目前automake支持三种目录层次：flat、shallow和deep。  

1)	flat指的是所有文件都位于同一个目录中。  

就是所有源文件、头文件以及其他库文件都位于当前目录中，且没有子目录。Termutils就是这一类。  

2)	shallow指的是主要的源代码都储存在顶层目录，其他各个部分则储存在子目录中。  

就是主要源文件在当前目录中，而其它一些实现各部分功能的源文件位于各自不同的目录。automake本身就是这一类。  

3)	deep指的是所有源代码都被储存在子目录中；顶层目录主要包含配置信息。  

就是所有源文件及自己写的头文件位于当前目录的一个子目录中，而当前目录里没有任何源文件。 GNU cpio和GNU tar就是这一类。  

flat类型是最简单的，deep类型是最复杂的。不难看出，我们的模拟需求正是基于第三类deep型，也就是说我们要做挑战性的事情：)。注：我们的测试程序是基于多线程的简单程序。  

### \#0x03 生成 Makefile 的来龙去脉
首先进入 project 目录，在该目录下运行一系列命令，创建和修改几个文件，就可以生成符合该平台的Makefile文件，操作过程如下：  

1)	运行autoscan命令  

2)	将configure.scan 文件重命名为configure.in，并修改configure.in文件  

3)	在project目录下新建Makefile.am文件，并在core、shell和lib目录下也新建makefile.am文件  

4)	在project目录下新建NEWS、 README、 ChangeLog 、AUTHORS文件  

5)	将/usr/share/automake-1.X/目录下的depcomp和complie文件拷贝到本目录下  

6)	运行aclocal命令  

7)  autoheader  

8)	运行autoconf命令  

9)	运行automake -a命令  

10)	运行./confiugre脚本  

11) make

可以通过图2看出产生Makefile的流程，如图所示：  

图2 生成Makefile流程图  
{:refdef: style="text-align: center;"}
![生成Makefile流程图](https://github.com/PythonPig/PythonPig.github.io/blob/master/images/利用autoconf和automake生成Makefile文件/生成Makefile流程图.gif?raw=true)
{: refdef}

#### Configure.in的八股文
当我们利用autoscan工具生成confiugre.scan文件时，我们需要将confiugre.scan重命名为confiugre.in文件。confiugre.in调用一系列autoconf宏来测试程序需要的或用到的特性是否存在，以及这些特性的功能。  

下面我们就来目睹一下confiugre.scan的庐山真面目：  
```
# Process this file with autoconf to produce a configure script.
AC_PREREQ(2.59)
AC_INIT(FULL-PACKAGE-NAME, VERSION, BUG-REPORT-ADDRESS)
AC_CONFIG_SRCDIR([config.h.in])
AC_CONFIG_HEADER([config.h])
# Checks for programs.
AC_PROG_CC
# Checks for libraries.
# FIXME: Replace `main' with a function in `-lpthread':
AC_CHECK_LIB([pthread], [main])
# Checks for header files.
# Checks for typedefs, structures, and compiler characteristics.
# Checks for library functions.
AC_OUTPUT
```
每个configure.scan文件都是以AC_INIT开头，以AC_OUTPUT结束。我们不难从文件中看出confiugre.in文件的一般布局：  
```
AC_INIT
 测试程序
 测试函数库
 测试头文件
 测试类型定义
 测试结构
 测试编译器特性
 测试库函数
 测试系统调用
AC_OUTPUT
```

上面的调用次序只是建议性质的，但我们还是强烈建议不要随意改变对宏调用的次序。  

现在就开始修改该文件：  
```
$mv configure.scan configure.in
$vim configure.in
```

修改后的结果如下：  
```
#                                -*- Autoconf -*-
# Process this file with autoconf to produce a configure script.
 
AC_PREREQ(2.59)
AC_INIT(test, 1.0, normalnotebook@126.com)
AC_CONFIG_SRCDIR([src/ModuleA/apple/core/test.c])
AM_CONFIG_HEADER(config.h)
AM_INIT_AUTOMAKE(test,1.0)
 
# Checks for programs.
AC_PROG_CC
# Checks for libraries.
# FIXME: Replace `main' with a function in `-lpthread':
AC_CHECK_LIB([pthread], [pthread_rwlock_init])
AC_PROG_RANLIB
# Checks for header files.
# Checks for typedefs, structures, and compiler characteristics.
# Checks for library functions.
AC_OUTPUT([Makefile
        src/lib/Makefile
        src/ModuleA/apple/core/Makefile
        src/ModuleA/apple/shell/Makefile
        ])
```
其中要将AC_CONFIG_HEADER([config.h])修改为：AM_CONFIG_HEADER(config.h), 并加入AM_INIT_AUTOMAKE(test,1.0)。由于我们的测试程序是基于多线程的程序，所以要加入AC_PROG_RANLIB，不然运行automake命令时会出错(AC_PROG_RANLIB is required if any libraries are built in the package)。在AC_OUTPUT输入要创建的Makefile文件名。  
由于我们是基于deep类型来创建makefile文件，所以我们需要在四处创建Makefile文件。即：project目录下，lib目录下，core和shell目录下。 


AC_PREREQ宏声明本文件要求的autoconf版本，autoscan自动生成  
AC_INIT宏用来定义软件的名称和版本等信息，”FULL-PACKAGE-NAME”为软件包名称，”VERSION”为软件版本号，”BUG-REPORT-ADDRESS”为BUG报告地址（一般为软件作者邮件地址）  
AC_CONFIG_SRCDIR宏用来侦测所指定的源码文件是否存在，来确定源码目录的有效性。工程中的任意源文件都可以，可以是main函数所在的c文件  
AM_CONFIG_HEADER宏用于生成config.h文件，以便autoheader使用 
AM_INIT_AUTOMAKE(PACKAGE,VERSION)这个是使用 Automake 所必备的宏，PACKAGE 是所要产生软件的名称，VERSION 是版本编号  
AC_PROG_CC用来指定编译器，如果不指定，选用默认gcc   
由于我们在程序中使用了读写锁，所以需要对库文件进行检查，即AC_CHECK_LIB([pthread], [main])，该宏的含义如下：  
{:refdef: style="text-align: center;"}
![AC_CHECK_LIB](https://github.com/PythonPig/PythonPig.github.io/blob/master/images/利用autoconf和automake生成Makefile文件/ac_check_lib.gif?raw=true)
{: refdef}

其中，LIBS是link的一个选项，详细请参看后续的Makefile文件。由于我们在程序中使用了读写锁，所以我们测试pthread库中是否存在pthread_rwlock_init函数。  

AC_PROG_RANLIB如果工程使用了库文件，则需要该宏  
AC_OUTPUT指定要创建的makefile

Autoconf提供了很多内置宏来做相关的检测，限于篇幅关系，我们在这里对其他宏不做详细的解释，具体请参看参考文献1和参考文献2，也可参看autoconf信息页。 



#### 实战Makefile.am
Makefile.am是一种比Makefile更高层次的规则。只需指定要生成什么目标，它由什么源文件生成，要安装到什么目录等构成。  

表一列出了可执行文件、静态库、头文件和数据文件，四种书写Makefile.am文件的一般格式。  

表1 Makefile.am一般格式  
{:refdef: style="text-align: center;"}
![Makefile.am一般格式](https://github.com/PythonPig/PythonPig.github.io/blob/master/images/利用autoconf和automake生成Makefile文件/Makefile.am一般格式.gif?raw=true)
{: refdef}

对于可执行文件和静态库类型，如果只想编译，不想安装到系统中，可以用noinst_PROGRAMS代替bin_PROGRAMS，noinst_LIBRARIES代替lib_LIBRARIES。  

Makefile.am还提供了一些全局变量供所有的目标体使用：  

表2 Makefile.am中可用的全局变量  
{:refdef: style="text-align: center;"}
![Makefile.am中可用的全局变量](https://github.com/PythonPig/PythonPig.github.io/blob/master/images/利用autoconf和automake生成Makefile文件/Makefile.am中可用的全局变量.gif?raw=true)
{: refdef}

在Makefile.am中尽量使用相对路径，系统预定义了两个基本路径：  

表3 Makefile.am中可用的路径变量  

{:refdef: style="text-align: center;"}
![Makefile.am中可用的路径变量](https://github.com/PythonPig/PythonPig.github.io/blob/master/images/利用autoconf和automake生成Makefile文件/Makefile.am中可用的路径变量.gif?raw=true)
{: refdef}

我们首先需要在工程顶层目录下（即project/）创建一个Makefile.am来指明包含的子目录：    

```
SUBDIRS=src/lib src/ModuleA/apple/shell src/ModuleA/apple/core 
CURRENTPATH=$(shell /bin/pwd)
INCLUDES=-I$(CURRENTPATH)/src/include -I$(CURRENTPATH)/src/ModuleA/apple/include 
export INCLUDES
```
由于每个源文件都会用到相同的头文件，所以我们在最顶层的Makefile.am中包含了编译源文件时所用到的头文件，并导出，见蓝色部分代码。 

我们将lib目录下的swap.c文件编译成libswap.a文件，被apple/shell/apple.c文件调用，那么lib目录下的Makefile.am如下所示：  
```
noinst_LIBRARIES=libswap.a
libswap_a_SOURCES=swap.c
INCLUDES=-I$(top_srcdir)/src/includ
```
细心的读者可能就会问：怎么表1中给出的是bin_LIBRARIES，而这里是noinst_LIBRARIES？这是因为如果只想编译，而不想安装到系统中，就用noinst_LIBRARIES代替bin_LIBRARIES，对于可执行文件就用noinst_PROGRAMS代替bin_PROGRAMS。对于安装的情况，库将会安装到$(prefix)/lib目录下，可执行文件将会安装到${prefix}/bin。如果想安装该库，则Makefile.am示例如下：  
```
bin_LIBRARIES=libswap.a
libswap_a_SOURCES=swap.c
INCLUDES=-I$(top_srcdir)/src/include
swapincludedir=$(includedir)/swap
swapinclude_HEADERS=$(top_srcdir)/src/include/swap.h
```
最后两行的意思是将swap.h安装到${prefix}/include /swap目录下。  

接下来，对于可执行文件类型的情况，我们将讨论如何写Makefile.am？对于编译apple/core目录下的文件，我们写成的Makefile.am如下所示：  
```
noinst_PROGRAMS=test
test_SOURCES=test.c 
test_LDADD=$(top_srcdir)/src/ModuleA/apple/shell/apple.o $(top_srcdir)/src/lib/libswap.a 
test_LDFLAGS=-D_GNU_SOURCE
DEFS+=-D_GNU_SOURCE
#LIBS=-lpthread
```
由于我们的test.c文件在链接时，需要apple.o和libswap.a文件，所以我们需要在test_LDADD中包含这两个文件。对于Linux下的信号量/读写锁文件进行编译，需要在编译选项中指明-D_GNU_SOURCE。所以在test_LDFLAGS中指明。而test_LDFLAGS只是链接时的选项，编译时同样需要指明该选项，所以需要DEFS来指明编译选项，由于DEFS已经有初始值，所以这里用+=的形式指明。从这里可以看出，Makefile.am中的语法与Makefile的语法一致，也可以采用条件表达式。如果你的程序还包含其他的库，除了用AC_CHECK_LIB宏来指明外，还可以用LIBS来指明。  

下面看看apple.o文件怎么生成：
如果你只想编译某一个文件，如apple.c，那么Makefile.am如何写呢？这个文件也很简单，写法跟可执行文件的差不多，如下例所示： 
```
noinst_PROGRAMS=apple
apple_SOURCES=apple.c
DEFS+=-D_GNU_SOURCE
```
这样写Makefile.am文件，最终会通过编译连接生成可执行文件，如果只想编译生成apple.o文件的话，这里需要欺骗automake(因为apple.c中没有main函数，生成可执行文件的话肯定报错)，假装要生成可执行文件apple，让它为我们生成依赖关系和执行命令。所以当你运行完automake命令后，然后修改apple/shell/下的Makefile.in文件，直接将LINK语句删除，这样就只会生成编译后的apple.o文件，而不会通过连接生成可执行文件apple,删除LINK语句如下：  

```
clean-noinstPROGRAMS:
    -test -z "$(noinst_PROGRAMS)" || rm -f $(noinst_PROGRAMS)
apple$(EXEEXT): $(apple_OBJECTS) $(apple_DEPENDENCIES) 
    @rm -f apple$(EXEEXT)
#$(LINK) $(apple_LDFLAGS) $(apple_OBJECTS) $(apple_LDADD) $(LIBS)
```
通过上述处理，就可以达到我们的目的。从图1中不难看出为什么要修改Makefile.in的原因，而不是修改其他的文件。  

如果文件很多，每个多要去修改Makefile.in的话工作量将会很大，另一个处理方法是直接将apple.c生成库文件libapple.a而不是apple.o。此时，Makefile.am如下：
```
noinst_LIBRARIES=libapple.a
libapple_a_SOURCES=apple.c
libapple_a_LIBADD=$(top_srcdir)/src/lib/libswap.a 
export INCLUDES
```

对应修改core/Makefile.am为：
```
noinst_PROGRAMS=test
test_SOURCES=test.c 
test_LDADD=$(top_srcdir)/src/ModuleA/apple/shell/libapple.a $(top_srcdir)/src/lib/libswap.a 
test_LDFLAGS=-D_GNU_SOURCE
DEFS+=-D_GNU_SOURCE
#LIBS=-lpthread
export INCLUDES
```



### 参考
* [例解 autoconf 和 automake 生成 Makefile 文件](https://www.ibm.com/developerworks/cn/linux/l-makefile/index.html)
* [使用automake](https://ipjmc.iteye.com/blog/1733377)
* [Linux下的Autoconf和AutoMake](https://www.linuxidc.com/Linux/2014-09/107014.htm)
* [6.2 Other things Automake recognizes](https://www.gnu.org/software/automake/manual/html_node/Optional.html)