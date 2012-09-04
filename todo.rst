10000 Miles TODO
==================

libs
------------------

core lang: [a, b, c]

Libs: Sys, User

Lib: Key -> Object

Lang Eco: DAG of Libs

Lib is Namespace

程序需要一种方法由lib名字找到lib所在的位置
lib的名字应该保持稳定

虚拟环境是一个自洽的DAG

因为复杂系统由不同实体完成，所以就需要合作。

要合作必须由接口。

Lib是一种接口

Python语法解析
-------------------

打开文件的几种方法
----------------------

package, module, __init__.py
------------------------------------
::

    jaime@westeros:~/source/longtalk/lib$ cat __init__.py
    class Helper:
        pass
    jaime@westeros:~/source/longtalk/lib$ cat settings.py
    import lib.Helper
    jaime@westeros:~/source/longtalk/lib$ cd ../
    jaime@westeros:~/source/longtalk$ python -c 'import lib.settings'
    Traceback (most recent call last):
      File "<string>", line 1, in <module>
      File "lib/settings.py", line 1, in <module>
        import lib.Helper
    ImportError: No module named Helper
    jaime@westeros:~/source/longtalk$ 

load files- the last step of importing
-----------------------------------------------
如果sys.path[0]是空字符串，则表示查找当前目录。python在搜索模块的时候，会遍历
sys.path中所有的path，os.path.join(path, module_name)，如果path为'', 则自然
就是在当前目录查找。

如果你把.py脚本文件作为参数传递给python解释器，那么sys.path[0]通常将是该文件
所在目录，即os.path.dirname(yourfile)，这就是为什么导入相对目录的模块会起作用。

sys.path[0]在 ``PySys_SetArgvEx`` 中设置::

    jaime@ideer:~/source/Python-2.6.7$ grep -rn PySys_SetArgv Python/ Modules/
    Python/frozenmain.c:48:    PySys_SetArgv(argc, argv);
    Python/sysmodule.c:1531:PySys_SetArgvEx(int argc, char **argv, int updatepath)
    Python/sysmodule.c:1635:PySys_SetArgv(int argc, char **argv)
    Python/sysmodule.c:1637:    PySys_SetArgvEx(argc, argv, 1);
    Modules/main.c:503:           so that PySys_SetArgv correctly sets sys.path[0]
    to ''*/
    Modules/main.c:508:    PySys_SetArgv(argc-_PyOS_optind, argv+_PyOS_optind);

import语句执行路径

imp模块是怎么回事
imp可以实现更灵活的模块导入

tracer.runfunc
---------------------------

sys.exc_type, raise statement
-----------------------------------------

sys.exc_type, sys.exc_xxx保存的每个frame捕获的最后一个异常，和当前frame相关，目的
是便利对异常进行分析，不用把异常传给分析函数。内层frame的异常并不会覆盖外层frame的异常::

    import sys
    import traceback

    def foo():
        try:
            raise IOError
        except IOError:
            pass
        bar()
        print 'foo', sys.exc_type, traceback.format_exc()

    def bar():
        try:
            raise KeyError
        except KeyError:
            pass
        print 'bar', sys.exc_type, traceback.format_exc()

    foo()

::

    09:19 jaime@oldtown ajourneyintopythonsource (master)$ python a.py 
    bar <type 'exceptions.KeyError'> Traceback (most recent call last):
      File "a.py", line 14, in bar
        raise KeyError
    KeyError

    foo <type 'exceptions.IOError'> Traceback (most recent call last):
      File "a.py", line 6, in foo
        raise IOError
    IOError

如果在bar函数里面也捕获了异常，假设类型位KeyError，那么在bar里面看到的sys.exc_type
就是KeyError, 而在foo函数中bar返回之后，foo看到的sys.exc_type还是IOError。

如果内层函数没有捕获过异常，则sys.exc_type仍回溯指向上一层的sys.exc_type。
保存frame为了分析使用。

空的raise语句作用是把sys.exc_type重新抛出。所以如果当前frame内没有捕获到异常，最终抛出的异常
可能会出乎你的意料。最佳做法是，总是显式的指定raise异常类型。

参考 Python/ceval.c set_exc_info, do_raise 函数的说明。

既然和frame相关，sys.exc_type就更是线程隔离的，每个线程有自己的异常状态。

Python/errors.c
~~~~~~~~~~~~~~~~~~~~~~~

设置当前线程异常为type, 抛出异常 ::

    void PyErr_Restore(PyObject *type, PyObject *value, PyObject *traceback)

PyErr_Occurred 判断是否有异常发生，简言之，如果当前线程的tstate->curexc_type不是NULL，
则python就认为有什么地方抛出异常了。

查看当前异常是否和给定的exc匹配， 用于except语句::
    
    void PyErr_ExceptionMatches(PyObject *exc) 

exc是异常class，可以是tuple，即多个class。Note: PyExceptionClass_Check,
PyObject_IsSubclass.

PyErr_Clear 清空异常信息，让系统认为没有异常发生。

其他都是一些辅助函数，便于抛出异常。NoMemory的异常比较有意思::

    PyObject *
    PyErr_NoMemory(void)
    {
        if (PyErr_ExceptionMatches(PyExc_MemoryError))
            /* already current */
            return NULL;

        /* raise the pre-allocated instance if it still exists */
        if (PyExc_MemoryErrorInst)
            PyErr_SetObject(PyExc_MemoryError, PyExc_MemoryErrorInst);
        else
            /* this will probably fail since there's no memory and hee,
               hee, we have to instantiate this class
            */
            //已经没有内存了，所以只有抛出一个class了事
            PyErr_SetNone(PyExc_MemoryError);

        return NULL;
    }

PyErr_NewException 生成新的异常类型？

PyErr_SyntaxLocation?

source code reloading
----------------------------
必须有一个dag才行

a.py::

    import b
    s = str(b.s)

b.py::

    s = "test"

reload b 对a不起作用，严格意义上来讲，a已经不依赖于b，运行中的a已经成功bootstrap，脱离了b。除非生成一个新的a。

这样的依赖关系dag没那么简单，只有清晰定义组件之间的封装接口，才可能做到完整的，在线live的reload。

logging探秘
-----------------

Py_NewInterpreter
----------------------------

Py_Initialize
--------------

Python协议
----------------
duck typing 是一种约定，好处就是便于伪装，只要你遵守规范，定义了特定的接口，
具体是什么类型倒是没有关系，去耦合

__init__
__call__
__iter__
__repr__
__next__

动态改变method函数定义的能力

setattr在什么情况下不起作用
-----------------------------


python thread
---------------------
Python VM指令集

http://docs.python.org/library/dis.html#python-bytecode-instructions

如果线程的实现有Python vm指令支持，想必会好很多，那可以说是真正native的python
thread。

