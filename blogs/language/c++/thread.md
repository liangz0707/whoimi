##临界区

在多线程当中进行互斥同步的锁。

可是使用windase.h当中的临时区锁进行控制：

**InitializeCriticalSection  此函数初始化一个临界区对象 **

void InitializeCriticalSection(  LPCRITICAL_SECTION *lpCriticalSection*);

参数：lpCriticalSection指向临界区对象的指针

这个进程负责分配一个临界区对象使用的内存，它可以通过声明类型的CRITICAL_SECTION的变量使用的内存。一旦一个临界区对象已被初始化，该进程的线程可以在EnterCriticalSection或LeaveCriticalSection函数指定对象，提供对共享资源的相互独占式访问。对于不同进程之间的类似线程同步，使用互斥对象。

一个临界区对象不能移动或复制。这一进程也绝不能修改该对象，但必须把它作为逻辑不透明来处理。只能使用由与Microsoft Win32 ® API提供的临界区功能，用来管理临界区对象。

在低内存的情况下，InitializeCriticalSection可能提出STATUS_NO_MEMORY异常。

**DeleteCriticalSection 删除关键节对象释放由该对象使用的所有系统资源。**

void WINAPI DeleteCriticalSection(_Inout_ LPCRITICAL_SECTION lpCriticalSection);

参数：*lpCriticalSection，*对关键节对象的指针。先前必须已将该对象初始化于InitializeCriticalSection对象中。

**线程锁的概念函数EnterCriticalSection和LeaveCriticalSection的用法**

使用结构CRITICAL_SECTION 需加入头文件#include “afxmt.h”

定义一个全局的锁 CRITICAL_SECTION的实例

```c
CRITICAL_SECTION cs;
InitializeCriticalSection(&cs);

//第一个线程
for(int i = 0; i<10; i++){      
  EnterCriticalSection(&cs);//加锁
	// TODO：逻辑
  LeaveCriticalSection(&cs);//解锁
}

//第二线程
for(int i = 0; i<10; i++){      
  EnterCriticalSection(&cs);//加锁
	// TODO：逻辑
  LeaveCriticalSection(&cs);//解锁
}

//使用完成
DeleteCriticalSection(&cs);
```



## 互斥锁和阻塞信号

SuspendThread 如果暂停了获取锁的进程：

```c
EnterCriticalSection(&cs);//加锁
// TODO:A
LeaveCriticalSection(&cs);//解锁
```

如果在A处暂停则可能造成死锁。

WaitForSingleObject是等待一个特定的对象编程发出信号的状态或者过时。

说明[WaitForSingleObject](https://msdn.microsoft.com/en-us/library/windows/desktop/ms687032(v=vs.85).aspx)

例子[WaitForSingleObject](https://msdn.microsoft.com/en-us/library/windows/desktop/ms686915(v=vs.85).aspx)



## 多线程编程之Windows同步方式

　本文来自[cnblogs.](http://www.cnblogs.com/kuliuheng/p/4062211.html)

​	在Windows环境下针对多线程同步与互斥操作的支持，主要包括四种方式：临界区（CriticalSection）、互斥对象（Mutex）、信号量（Semaphore）、事件对象（Event）。下面分别针对这四种方式作说明：

**（1）临界区（CriticalSection）**

　　每个进程中访问临界资源的那段代码称为临界区（临界资源是一次仅允许一个进程使用的共享资源）。每次只准许一个进程进入临界区，进入后不允许其他进程进入。不论是硬件临界资源，还是软件临界资源，多个进程必须互斥地对它进行访问。Windows环境下临界区的基本操作有以下几个：

```c
CRITICAL_SECTION CriticalSection;
InitializeCriticalSection(&CriticalSection);
EnterCriticalSection(&CriticalSection);
LeaveCriticalSection(&CriticalSection);
DeleteCriticalSection(&CriticalSection);
```

**（2）互斥对象（Mutex）**

　　在编程中，引入了对象互斥对象（也叫互斥锁）的概念，来保证共享数据操作的完整性。每个对象都对应于一个可称为“互斥锁”的标记，这个标记用来保证在任一时刻，只能有一个线程访问该对象。互斥对象的操作接口有以下几个：

```c
CreateMutex
OpenMutex
ReleaseMutex
```

 　　在使用互斥对象的时候借助WaitforSingleObject，例如：

```c
WaitForSingleObject(/*...*/);
    do_something();
ReleaseMutex(/*...*/);
```

 **（3）信号量（Semaphore）　　**

　　信号量有时被称为信号灯，是在多线程环境下使用的一种设施，它负责协调各个线程，以保证它们能够正确、合理的使用公共资源。也是操作系统中用于控制**进程同步互斥**的量。信号量分为单值和多值两种，前者只能被一个线程获得，后者可以被若干个线程获得。与互斥对象相比，信号量就好比是可以容纳N多个人的房子允许多个人同时进入（数量有限制而已），而互斥对象就只能容纳一个人的小房子，同一时刻只能一个人使用。

　　Windows环境下的信号量操作接口包括：

```c
CreateSemaphore
OpenSemaphore
ReleaseSemaphore
```

 　　信号量的使用方式与互斥对象差不多，只不过在初始化的时候需要指定信号的个数：

```c
WaitForSingleObject(/*...*/);
    do_something();
ReleaseSemaphore(/*...*/);
```

 **（4）事件对象（Event）**

　　 Event对象是Windows下面很有趣的一种锁结果。从某种意义上说，它和互斥锁很相近，但是又不一样。因为在线程获得锁的使用权之前，常常需要某一个线程（可能是主线程也可能是其他线程）调用SetEvent设置一下才行。关键是，在线程结束之前，我们也不清楚当前线程获得Event之后执行到哪了。所以使用起来，要特别小心。常用的Event对象操作有：

```c
CreateEvent
OpenEvent
PulseEvent
ResetEvent
SetEvent
```

 　　主线程一般可以这样做：

```c
CreateEvent(/*...*/);    // 创建事件对象
SetEvent(/*...*/);       // 设置信号
WaitForMultiObjects(hThread, /*...*/);    // 等待线程结束
CloseHandle(/*...*/);    // 关闭线程句柄
```

 　　而被启动的线程一般要等待某个事件再进行动作：

```c
while(1){
    WaitForSingleObject(/*...*/);    // 等待事件
    /*...*/
}
```

 

**总结：**
（1）关于[临界区](http://msdn.microsoft.com/en-us/library/ms686908(v=VS.85).aspx)、[互斥区](http://msdn.microsoft.com/en-us/library/ms686927(v=VS.85).aspx)、[信号量](http://msdn.microsoft.com/en-us/library/ms686946(v=VS.85).aspx)、[E](http://msdn.microsoft.com/en-us/library/ms686915(v=VS.85).aspx)[vent](http://msdn.microsoft.com/en-us/library/ms686915(v=VS.85).aspx)在msdn上均有示例代码；

（2）一般来说，使用频率上**信号量 > 互斥对象 > 临界区 > 事件对象**

（3）信号量可以实现其他三种锁的功能，学习上应有所侧重

（4）纸上得来终觉浅，多实践才能掌握它们之间的区别 *    *