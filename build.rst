0 Miles
==============

线程支持
--------
禁用线程支持::
    
    ./configure --without-thread


打开调试开关
------------
构建debug版本，让系统在运行时打出调试信息::

    ./configure --with-pydebug

参见Misc/SpecialBuilds.txt Py_DEBUG


禁用动态扩展加载
----------------------
``HAVE_DYNAMIC_LOADING`` 这个宏决定是否支持动态扩展模块加载::

    #undef HAVE_DYNAMIC_LOADING


动态编译
------------------
是否将python本身编译成共享库，libpython2.6.a还是libpython2.6.so。第三方程序需要链接libpython时，
有的会要求使用动态链接的版本，如mod_python::

    ./configure --enable-shared

    ldd python

如果是动态编译，make install之后必须设置动态库加载路径::

    /etc/ld.so.config/
    ldconfig

#FIXME: -fPIC http://en.wikipedia.org/wiki/Position-independent_code

此配置项和c ext的so模块没有关系。
