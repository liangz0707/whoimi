“这篇博客的内容来自我原来开源中国的博客这里修正了一下~因为原来直接拷贝的ipython的格式代码是有问题的~”

下面的内容是直接从我的ipython notebook中摘录出来的所以可能不是很清楚，如果想查看完整版本以及例子，可以访问：

[git地址](git@github.com:liangz0707/WTF-makefile.git)

[下载地址](https://github.com/liangz0707/WTF-makefile/archive/master.zip)

如果不能查看ipython notebook 可以直接打开其中的lession.html 结果是一样的

## 首先讲解编译器等内容
What is Compiler？

Compiler=编译器，就是将某种代码编译成机器语言，或者说编译成能够由处理器直接执行的“代码”。一个程序员会以一种语言在编辑器当中编写语句。例如：c、Lisp。编写完成后生成的文件就是源代码。然后程序员运行具体的编译器，把上述文件名作为参数进行编译。

执行过程中，编译器会依照语法顺序，逐一对语句进行解析。最终生成输出代码。一般的编译器的输出结果叫做object code或者object module（这里的object和面向对象中的object不同）。这里的object code是机器语言。

在传统的操作系统当中，在编译之后往往还需要额外的步骤，因为当有多于一个Object Code时 他们的指令、和数据之间存在关系，所以需要进行链接，得到的结果是。 load module。

## 下面使用具体的编译器GCC来讲解
GCC是c语言的编译器之一：

gcc is the "GNU" C Compiler, and g++ is the "GNU C++ compiler

我们编写一个hello.c文件（见目录）：
```c
#include "iostream.h"
int main()
{
cout &lt;&lt; "Hello\n";
}
```

执行如下命令，输出的结果就是可执行的机器码（.exe可执行程序），默认文件名是a.exe

主要g++编译的源文件中头文件的引用一般为#include,并且要注意使用命名空间。

```shell
g++ hello.c -o hello
```

## Makefiles的使用
在实际的编译过程中，逐个的编译源代码过于冗杂，尤其是当你需要包含很多源文件时，你不得不每一次都要打字输入。

而是用Makefile就是为了简化很多源文件的编译过程。 例如我们有很多c问见需要编译:
<ul>
	<li>main.cpp</li>
	<li>hello.cpp</li>
	<li>factorial.cpp</li>
	<li>functions.h 以上的文件都是通过头文件串联起来的</li>
</ul>
如果需要手动的编译，如下非常麻烦

```shell
g++ main.cpp hello.cpp factorial.cpp -o hello
```

执行上面得到的文件

```shell
hello
```

###编译的过程包含了：
<ul>
	<li>把source code转换成object files</li>
	<li>将不同的object files连接成可执行的机器码（.exe) 下面使用最简单的Makefile 来进行编译，代替手工过程</li>
</ul>
基本的Makefile组成如下：

<code><span style="color: #ff0000; font-family: Consolas;">target: dependencies</span></code>

<code><span style="color: #ff0000; font-family: Consolas;">[tab]system command</span></code>

‘target’可以理解为要编译的目标或任务（分号后的内容），dependencies表示要完成这个目标所需要的前提。编写成我们需要的格式如下：

<code><span style="color: #ff0000; font-family: Consolas;">all:     </span></code>

<code><span style="color: #ff0000; font-family: Consolas;">    g++ main.cpp hello.cpp factorial.cpp -o hello</span></code>

从makefile文件中可以看出我们的目的是‘all’，这是makefiles默认的目标。make工具在没有特殊声明的时候会有限执行‘all’，我们看到all是没有依赖的，所以可以安全的执行。
<h3>那么如何使用依赖关系呢？</h3>
有时候我们会使用不同的‘target’。因为当我们修改了一个文件的时候，不希望把所有的文件都进行重新编译。

例如以下的例子：
```makefiles
all: hello
hello: main.o factorial.o hello.o     
    g++ main.o factorial.o hello.o -o hello
main.o: main.cpp     
    g++ -c main.cpp
factorial.o: factorial.cpp     
    g++ -c factorial.cpp
hello.o: hello.cpp     
    g++ -c hello.cpp
clean:     
    del *o hello.exe
```

相当于把编译的过程（上述的两部，编译、连接）拆分开。

我们看到all只有一个依赖，而没有命令，这是为了让make能够正确的执行。all必须执行才能完成。

对于每一个可用的目标所有的依赖都会被搜索，如果找到则执行。

我们还看到了一个clean的目标,他可以快速的清除所有的object和可执行程序。

### 在makefiles中使用变量和注释
例子如下：

```makefiles
# I am a comment, and I want to say that the variable CC will be

# the compiler to use. CC=g++

# Hey!, I am comment number 2. I want to say that CFLAGS will be the

# options I'll pass to the compiler.

CFLAGS=-c -Wall

all: hello

hello: main.o factorial.o hello.o    

    $(CC) main.o factorial.o hello.o -o hello

main.o: main.cpp     

    $(CC) $(CFLAGS) main.cpp

factorial.o: factorial.cpp     

    $(CC) $(CFLAGS) factorial.cpp

hello.o: hello.cpp     

    $(CC) $(CFLAGS) hello.cpp

clean:     

    del *o hello.exe
```
如上所示，使用$(VAR)就可以轻松的访问变量。


[back](../../index.md)