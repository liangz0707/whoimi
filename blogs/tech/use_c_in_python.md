# 在python中使用c/c++模块

在python当中使用c++的方式有很多主要可以使用

主要可以参考下面两个链接：

该方法是在源代码级别进行调用，需要重新调用源代码，生成的c模块也可以被所有的python代码调用

[官方扩展说明](https://docs.python.org/2/extending/extending.html)

该链接包含了众多的方法，可以选择一个进行使用

[多个python调用c/c++的方法](http://intermediate-and-advanced-software-carpentry.readthedocs.io/en/latest/c++-wrapping.html#manual-wrapping)


下面我将要介绍的是第二个链接中，手动配置模块的过程，不使用外部的库。并且完成的模块可以被所有python随时调用。
1. 首先需要写一个c语言的文件
2. 这个文件的名字是<strong>spammodule.c</strong>就是我们需要的源文件。

```c
#include<python2.7/Python.h>//头文件时固定的
//该方法的参数是固定的，方法名并非python中要调用的方法名
static PyObject*
spam_system(PyObject* self, PyObject *args) {
      const char *command;
      int sts;
      // 检查类型并进行能类型转换,如果非0则表示转换称重结果保存在出入的地址当>  
      // 中。否则返回0。返回的指针时不可以修改的所以声明成了const
      if(!PyArg_ParseTuple(args,"s",&command))
         return NULL;
     //系统调用
     sts = system(command);
     //简历一个python的返回类型
     return Py_BuildValue("i",sts);
     /*
      * 如果是空返回值：
      * Py_INCREF(Py_None);
      * return Py_None;
     */
}

static PyMethodDef SpamMethods[] = {
    // 这里第一个表示python中的方法名，第二个表示对应c的方法，后面固定
    {"system",  spam_system, METH_VARARGS, "Execute a shell command."},
    {NULL, NULL, 0, NULL}        /* Sentinel */
};
 
DL_EXPORT(void) initspam(void)
{
    // 这里表示把模块spam中加入SpamMethods当中所有的方法
   // 之后在python中就可以使用import spam  调用spam.system
      (void)Py_InitModule("spam", SpamMethods);
}
```

源文件每个方法的含义可以参考链接当中的说明,下面需要将这个源文件安装到python中，和链接一中编译源文件不同，这个方法是将模块直接安装到python中，下面时安装代码
```python
from distutils.core import setup, Extension
#表示安装的模块名和源文件名
extension_mod = Extension("spam",["spammodule.c"])
#表示进行安装
setup(name="spam",ext_modules=[extension_mod])
```

下面是在python中调用我们安装好的内容

```python
# coding:utf-8
__author__ = 'liangz14'

import spam
status = spam.system("ls -l")
if __name__ == "__main__":
    print status
```


[back to list](../../index.md)