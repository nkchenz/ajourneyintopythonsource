
线程支持

--without-thread 禁用线程支持


调试

构建debug版本，系统在运行时会打出许多调试信息

./configure --with-pydebug


builtin模块

你可以在Modules/Setup.dist文件中指定将哪些模块内置到python可执行程序库中。
如果Setup文件不存在，make命令会将Setup.dist复制为Setup文件。但是一旦存在，
则不会在复制，故修改Setup.dist后，必须手动复制为Setup方能生效，或者你可以
直接修改Setup文件。


动态扩展加载
dynamic_loading是否支持加载动态扩展模块，即C代码写的so动态库


动态编译libpython

--enable-shared
是否将python本身编译成共享库，libpython2.6.a还是libpython2.6.so。
注意这和python是否支持加载动态扩展无关。第三方程序需要链接libpython时，有的
会要求使用动态链接的版本，如mod_python。
带fPIC编译标志的静态编译，效果和动态编译相当。？

    ldd python

如果是动态编译，make install之后必须设置动态库加载路径
/etc/ld.so.config/
ldconfig

