##锁

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

