37 Miles 模块
===============

builtin的函数
-----------------------
__builtin__ 模块Python/bltinmodule.c

builtin_methods

dir, I saw you!


builtin的模块列表
-------------------------------
你可以在Modules/Setup.dist文件中指定将哪些模块内置到python可执行程序库中。
如果Setup文件不存在，make命令会将Setup.dist复制为Setup文件。但是一旦存在, 则
不会在复制，故修改Setup.dist后，必须手动复制为Setup方能生效，或者你可以直接
修改Setup文件。

    sys.builtin_module_names

进一步分析如何完成链接

sys模块
-------
Python/sysmodule.c
sys.path


os模块
------
对于linux来说，os模块的大多数操作是从posix模块中导入的，后者代码在
Modules/posixmodule.c::

    _names = sys.builtin_module_names

    if 'posix' in _names:
        name = 'posix'
        linesep = '\n'
        from posix import *
        try:
            from posix import _exit
        except ImportError:
            pass
        import posixpath as path

        import posix
        __all__.extend(_get_exports_list(posix))
        del posix

其他系统有nt，os2等模块，这些才是真正的底层实现，os模块只是提供一个跨平台的
封装。


sys.path[0] python怎样找到你的模块
--------------------------------------
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


PYTHONHOME和PYTHONPATH
-----------------------
calculate_path


多版本python的一些信息
--------------------------
python在启动的时候，会根据PYTHONHOME查看自身bin所在位置，从而推断出相应
版本的标准lib所在位置。

python运行需要的信息如下：
python      可执行文件
系统标准lib 用.py写的自带模块，.so扩展
用户模块    用户编写的.py文件
第三方包 你的程序中导入的第三方模块  

知道了以上信息，就可以构建一个完整的python运行环境了。


sys.executable来自何方
------------------------
Get_Path函数

Modules/getpath.c

module_search_path最终将成为sys.path

一般情况下，sys.executable都会被正确设置，如交互模式，手动启动python命令执行
文件。如果你在程序里嵌入Python，则可能有问题，虽然影响不大。


import语句执行路径
--------------------------


imp模块是怎么回事
-------------------
imp可以实现更灵活的模块导入


建立socket连接
-----------------------

    socket
       bind
          listen
          connect


解释器和c函数交互
-----------------------------
C扩展里定义的函数，怎么和python VM结合起来？


10000 Miles
===========
Py_NewInterpreter
----------------------------
