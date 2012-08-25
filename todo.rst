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
保存frame为了分析只用

空的语句作用是把sys.exc_type重新抛出。所以如果当前frame内没有捕获到异常，最终抛出的异常
可能会出乎你的意料。最佳做法是，总是显式的指定raise异常类型。

参考 Python/ceval.c set_exc_info, do_raise 函数的说明。

todo: 异常处理查找机制


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
