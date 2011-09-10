
1 Miles Source Coding
======================

代码结构
---------
Python      python解释器本身

Object      内置对象操作定义

Modules     C语言写的模块

Lib         python语言写的模块


main函数在哪里
----------------------
Modules/python.c main:
    Modules/main.c Py_Main: python命令行参数的解析
        Modules/pythonrun.c Py_Initialize 初始化解释器


添加自定义的.h或.c文件
----------------------------
Include/Python.h是python头文件的总索引，具体的常量，数据结构定义在Include目录
下其他文件中。因为它已被其他python c源代码文件包含，所以只需要将自定义的.h文件
添加到其最后，就可以引用到我们定义的变量：

    #include "foobar.h"

添加.c文件需要修改 ``Makefile.pre.in`` ，告诉python需要编译对应的obj文件。


c代码与python代码之间参数传递
------------------------------
PyArg_ParseTuple()

PyArg_ParseTupleAndKeywords()

PyArg_Parse()

为了理解源码中出现的C函数是干什么用的，你最好做一下准备工作，阅读 `The Python/C API`_ 。

.. _`The Python/C API`: http://docs.python.org/release/2.6.7/c-api/index.html


关于DECREF, INCREF
-------------------
+ 如果原函数已不再持有该object，ref为0，那么你需要INCREF，你是新的owner

+ 如果原函数仍然持有该object，ref肯定不为0，那么不需要INCREF，你只是暂时
  借用一下

+ 还有一种情况需要考虑，在借用过程中，如果原函数或者另外控制流有DECREF即
  释放该对象的可能，那么你最好INCREF，避免你的对象在操作过程中被别人删除

以上只是大致的说明，保险的办法是大量参考已有代码的做法。

