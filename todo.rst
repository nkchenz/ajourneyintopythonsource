10000 Miles
===========

__init__.py 在什么时候被执行
--------------------------------
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


Py_NewInterpreter
----------------------------

Py_Initialize
--------------

python缩进语法怎么实现
-----------------------


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
