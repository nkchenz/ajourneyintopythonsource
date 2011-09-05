
Build
=======

Makefile
---------
真正起作用的Makefile是Makefile.pre.in，它是configure脚本生成
Makefile的模板


线程支持
--------
``./configure --without-thread`` 禁用线程支持


打开调试开关
------------
构建debug版本，系统在运行时打出调试信息

    ./configure --with-pydebug


动态扩展加载
------------
``HAVE_DYNAMIC_LOADING`` 这个宏决定是否支持加载动态扩展模块，即C代码写的so动态库


动态编译libpython
------------------
``./configure --enable-shared``
是否将python本身编译成共享库，libpython2.6.a还是libpython2.6.so。
注意这和python是否支持加载动态扩展无关。第三方程序需要链接libpython时，有的
会要求使用动态链接的版本，如mod_python。
带fPIC编译标志的静态编译，效果和动态编译相当。？

    ldd python

如果是动态编译，make install之后必须设置动态库加载路径
/etc/ld.so.config/
ldconfig

