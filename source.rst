
1 Miles Source Coding
======================

代码结构
---------
Python      python解释器本身

Object      内置对象操作定义

Modules     C语言写的模块

Lib         python语言写的模块


python的main函数在哪
----------------------
Modules/python.c main:
    Modules/main.c Py_Main: python命令行参数的解析
        Modules/pythonrun.c Py_Initialize 初始化解释器


如何添加自定义的.h或.c文件
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

参阅The Python/C API Release 2.6.7

