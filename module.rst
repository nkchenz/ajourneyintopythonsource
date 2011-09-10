37 Miles
===============

从urllib2.urlopen到socket
----------------------------
urlopen::

    _opener = None
    def urlopen(url, data=None, timeout=socket._GLOBAL_DEFAULT_TIMEOUT):
        global _opener
        if _opener is None:
            _opener = build_opener()
        return _opener.open(url, data, timeout)

urllib2.urlopen共用一个模块变量_opener，也就是install_opener的那个，
搞并发的同学注意了，未知不同请求之间会否相互影响。

urlopen -> build_opener -> OpenerDirector.open, _open, __call_chain__ -> HTTPHandler.http_open ->
AbstractHTTPHandler->do_open -> HTTPConnection.request, _send_request,
send, connect

经过漫长的。。。，鄙人走马观花，自由行的同学可以深入研究:)
终于看到了socket.create_connection, Lib/httplib.py class HTTPConnection::

    def connect(self):
        """Connect to the host and port specified in __init__."""
        self.sock = socket.create_connection((self.host,self.port),
                                             self.timeout)
    ....
    
    def send(self, str):
        """Send `str' to the server."""
        if self.sock is None:
            if self.auto_open:
                self.connect()
            else:
                raise NotConnected()

Lib/socket.py::

        def create_connection(address, timeout=_GLOBAL_DEFAULT_TIMEOUT):
            ....
            msg = "getaddrinfo returns an empty list"
            host, port = address
            for res in getaddrinfo(host, port, 0, SOCK_STREAM):
                af, socktype, proto, canonname, sa = res
                sock = None
                try:
                    sock = socket(af, socktype, proto)
                    if timeout is not _GLOBAL_DEFAULT_TIMEOUT:
                        sock.settimeout(timeout)
                    sock.connect(sa)
                    return sock

在这里，通过getaddrinfo完成dns解析，建了一个socket，sock是内置socketobject类型，
从sock.connect开始，你就潜入C代码的世界了，在 Modules/socketmodule.c +2027::

    static PyObject *
    sock_connect(PySocketSockObject *s, PyObject *addro)
    {
        sock_addr_t addrbuf;
        int addrlen;

费了这半天劲，其实有个简单的方法，你就可以得到这整个的调用路径，yes，万能的raise::

    jaime@ideer:~/source/Python-2.6.7$ git df
    diff --git a/Lib/socket.py b/Lib/socket.py
    index e4f0a81..2a59dd9 100644
    --- a/Lib/socket.py
    +++ b/Lib/socket.py
    @@ -552,6 +552,7 @@ def create_connection(address, timeout=_GLOBAL_DEFAULT_TIMEOUT):
                 if timeout is not _GLOBAL_DEFAULT_TIMEOUT:
                     sock.settimeout(timeout)
                 sock.connect(sa)
    +            raise
                 return sock
     
             except error, msg:

    jaime@ideer:~/source/Python-2.6.7$ ./python
    Python 2.6.7 (r267:88850, Sep  8 2011, 22:55:29) 
    [GCC 4.5.2] on linux2
    Type "help", "copyright", "credits" or "license" for more information.
    >>> import urllib2
    >>> urllib2.urlopen('http://douban.com')
    Traceback (most recent call last):
      File "<stdin>", line 1, in <module>
      File "/home/chenz/source/Python-2.6.7/Lib/urllib2.py", line 126, in urlopen
        return _opener.open(url, data, timeout)
      File "/home/chenz/source/Python-2.6.7/Lib/urllib2.py", line 391, in open
        response = self._open(req, data)
      File "/home/chenz/source/Python-2.6.7/Lib/urllib2.py", line 409, in _open
        '_open', req)
      File "/home/chenz/source/Python-2.6.7/Lib/urllib2.py", line 369, in _call_chain
        result = func(*args)
      File "/home/chenz/source/Python-2.6.7/Lib/urllib2.py", line 1181, in http_open
        return self.do_open(httplib.HTTPConnection, req)
      File "/home/chenz/source/Python-2.6.7/Lib/urllib2.py", line 1153, in do_open
        h.request(req.get_method(), req.get_selector(), req.data, headers)
      File "/home/chenz/source/Python-2.6.7/Lib/httplib.py", line 914, in request
        self._send_request(method, url, body, headers)
      File "/home/chenz/source/Python-2.6.7/Lib/httplib.py", line 951, in _send_request
        self.endheaders()
      File "/home/chenz/source/Python-2.6.7/Lib/httplib.py", line 908, in endheaders
        self._send_output()
      File "/home/chenz/source/Python-2.6.7/Lib/httplib.py", line 780, in _send_output
        self.send(msg)
      File "/home/chenz/source/Python-2.6.7/Lib/httplib.py", line 739, in send
        self.connect()
      File "/home/chenz/source/Python-2.6.7/Lib/httplib.py", line 720, in connect
        self.timeout)
      File "/home/chenz/source/Python-2.6.7/Lib/socket.py", line 555, in create_connection
        raise
    TypeError: exceptions must be old-style classes or derived from BaseException, not NoneType
    >>> 


urllib2.py OpenerDirector的open函数::

        def open(self, fullurl, data=None, timeout=socket._GLOBAL_DEFAULT_TIMEOUT):
                # accept a URL or a Request object
                if isinstance(fullurl, basestring):
                    req = Request(fullurl, data)
                else:
                    req = fullurl
                    if data is not None:
                        req.add_data(data)

                req.timeout = timeout
                protocol = req.get_type()

                # pre-process request
                meth_name = protocol+"_request"
                for processor in self.process_request.get(protocol, []):
                    meth = getattr(processor, meth_name)
                    req = meth(req)

                response = self._open(req, data)

                # post-process response
                meth_name = protocol+"_response"
                for processor in self.process_response.get(protocol, []):
                    meth = getattr(processor, meth_name)
                    response = meth(req, response)

                return response

涵盖了一个http请求的全部过程，创建Request对象，获得协议类型，对请求进行预处理如
header，认证等，打开连接，处理响应，错误处理等，值得细究。


urllib2中的重定向
---------------------
http_response负责对服务器响应进行处理。如果状态码如果不是2xx，则启动错误处理机制::

    class HTTPErrorProcessor(BaseHandler):
        """Process HTTP error responses."""
        handler_order = 1000  # after all other processing

        def http_response(self, request, response):
            code, msg, hdrs = response.code, response.msg, response.info()

            # According to RFC 2616, "2xx" code indicates that the client's
            # request was successfully received, understood, and accepted.
            if not (200 <= code < 300):
                response = self.parent.error(
                    'http', request, response, code, msg, hdrs)

            return response

        https_response = http_response


3xx重定向指令由HTTPRedirectHandler负责，具体函数为http_error_3xx，主要做一些外围性
检查，分析获取重定向的地址，检测协议和循环重定向。如果一切ok，则调用redirect_request
生成新的Request对象，传给parent opener执行这个新req。一切又回到了开始。


start_response和exc_info
------------------------------

`WSGI`_ 规定了两个函数, write 和start_response::

    def start_response(status, response_headers, exc_info=None):

start_response返回write函数。这是为了和惯于用print类的应用进些兼容。
wsgi的application默认返回iterable，含有所有要输出的内容，server遍历它，
完成真正的输出::


 result = application(environ, start_response)
    try:
        for data in result:
            if data:    # don't send headers until body appears
                write(data)
        if not headers_sent:
            write('')   # send headers now if body was empty
    finally:
        if hasattr(result, 'close'):
            result.close()

write函数一旦被调用，就会自动激活header的输出，所以调用write是你改变header的
最后机会。

exc_info主要用于对异常进些处理，pep333中的示例代码::

    try:
        # regular application code here
        status = "200 Froody"
        response_headers = [("content-type", "text/plain")]
        start_response(status, response_headers)
        return ["normal body goes here"]
    except:
        # XXX should trap runtime issues like MemoryError, KeyboardInterrupt
        #     in a separate handler before this bare 'except:'...
        status = "500 Oops"
        response_headers = [("content-type", "text/plain")]
        start_response(status, response_headers, sys.exc_info())
        return ["error body goes here"]

异常发生时，如果：

* 200 OK没有被发送，没有调用过write，或者应用返回的iteralbe内容server还没有开始
  发送，总之，header没有发出，此时还有挽救的余地，将状态码改为500，忽略掉exc_info，
  用户自定义的错误信息，debug堆栈信息可以在error body里面输出。

* 200 OK这个header已经被server发送给客户端，已经发送了部分后续body内容，此时程序抛出
  异常，application探测到错误，怎么办？再发送500 Oops状态码也无济于事，wsgi server
  能做的只是raise exc_info，把事情搞大，捅到上层去。wsgi规定用户不可以捕捉带有exc_info
  信息的start_response抛出的异常。

start_response对这两种情况提供了一种统一的处理方式。在cgi环境里运行的wsgi start_response::

  def start_response(status, response_headers, exc_info=None):
        if exc_info:
            try:
                if headers_sent:
                    # Re-raise original exception if headers sent
                    raise exc_info[0], exc_info[1], exc_info[2]
            finally:
                exc_info = None     # avoid dangling circular ref
        elif headers_set:
            raise AssertionError("Headers already set!")

        headers_set[:] = [status, response_headers]
        return write


复杂的代码，不知道异常抛出时的准确状态，此为start_response exc_info的目的，可以用try except
把application的整个逻辑保护起来。或者你本就不该写复杂的代码？笑:) 或许你可以精巧的构造异常
处理代码，将header是否发送区分开来？

http协议的状态码status 200表示资源找到，但是后续处理出问题，怎么办？是否可以加一些位于最后的header，
表示请求成功完成？这样即使header已经发送，也可以做些别的措施暗示请求出错。content-length
是否起到了这样的作用？这也许是属于不同层的问题。

是否可以改变应用逻辑，全部处理完毕后一起发送header和body？区分应用相关，数据量大或长时间的应用
如何处理？stream？

.. _`WSGI`: http://www.python.org/dev/peps/pep-0333/

builtin的函数在哪
-----------------------
__builtin__ 模块对应的c文件是Python/bltinmodule.c::

    static PyMethodDef builtin_methods[] = {
        {"__import__",      (PyCFunction)builtin___import__, METH_VARARGS | METH_KEYWORDS, import_doc},
        {"abs",             builtin_abs,        METH_O, abs_doc},
        ...
        {"dir",             builtin_dir,        METH_VARARGS, dir_doc},
        {"divmod",          builtin_divmod,     METH_VARARGS, divmod_doc},
     
dir, I saw you! 这就是python dir函数的入口，对应的c代码为builtin_dir::

        static PyObject *
        builtin_dir(PyObject *self, PyObject *args)
        {
            PyObject *arg = NULL;

            if (!PyArg_UnpackTuple(args, "dir", 0, 1, &arg))
                return NULL;
            return PyObject_Dir(arg);
        }

进行简单的参数处理，获得参数object的指针，然后调用该object自身的dir处理函数，simple。
至于PyObject_Dir如何工作，则为后话了。现在不妨翻看一下其他的builtin函数代码。

PyArg_UnpackTuple 参数分析

+ args 是从python上层传过来的参数tuple
  
+ "dir" 用于出错时显示哪个函数::

    >>> dir(1, 2)
    Traceback (most recent call last):
    File "<stdin>", line 1, in <module>
    TypeError: dir expected at most 1 arguments, got 2

+ 0表示参数个数最少为0，1表示最多为1
  
+ &arg 提取到的参数存放在这里


METH_O 表示该函数只有一个参数，METH_VARARGS表示参数个数可变，具体定义在Include/methodobject.h::

    jaime@ideer:~/source/Python-2.6.7$ grep -rn METH_O Include/
    Include/methodobject.h:53:#define METH_OLDARGS  0x0000
    Include/methodobject.h:56:/* METH_NOARGS and METH_O must not be combined with the flags above. */
    Include/methodobject.h:58:#define METH_O        0x0008
    jaime@ideer:~/source/Python-2.6.7$ grep -rn METH_O Python/
    ...
    Python/ceval.c:3730:        if (flags & (METH_NOARGS | METH_O)) {
    Python/ceval.c:3736:            else if (flags & METH_O && na == 1) {
    jaime@ideer:~/source/Python-2.6.7$ 

在builtin_methods数组中只是声明了一下，运行时的参数检查在Python/ceval.c +3729 完成::


    PCALL(PCALL_CFUNCTION);
    if (flags & (METH_NOARGS | METH_O)) {
        PyCFunction meth = PyCFunction_GET_FUNCTION(func);
        PyObject *self = PyCFunction_GET_SELF(func);
        if (flags & METH_NOARGS && na == 0) {
            C_TRACE(x, (*meth)(self,NULL));
        }
        else if (flags & METH_O && na == 1) {
            PyObject *arg = EXT_POP(*pp_stack);
            C_TRACE(x, (*meth)(self,arg));
            Py_DECREF(arg);
        }
        else {
            err_args(func, flags, na);
            x = NULL;
        }
    }

如果定义了METH_NOARGS或METH_O，但是参数个数na又不为0或1，则通过err_args报错。

Python/ceval.c +3661::

    static void
    err_args(PyObject *func, int flags, int nargs)
    {
        if (flags & METH_NOARGS)
            PyErr_Format(PyExc_TypeError,
                         "%.200s() takes no arguments (%d given)",
                         ((PyCFunctionObject *)func)->m_ml->ml_name,
                         nargs);
        else
            PyErr_Format(PyExc_TypeError,
                         "%.200s() takes exactly one argument (%d given)",
                         ((PyCFunctionObject *)func)->m_ml->ml_name,
                         nargs);
    }


Hello, exception! 第一个异常
------------------------------

Modules/posixmodule.c +6313::

    static PyObject *
    posix_open(PyObject *self, PyObject *args)
    {
        char *file = NULL;
        int flag;
        int mode = 0777;
        int fd;

    #ifdef MS_WINDOWS
        if (unicode_file_names()) {
            PyUnicodeObject *po;
            if (PyArg_ParseTuple(args, "Ui|i:mkdir", &po, &flag, &mode)) {
                Py_BEGIN_ALLOW_THREADS
                /* PyUnicode_AS_UNICODE OK without thread
                   lock as it is a simple dereference. */
                fd = _wopen(PyUnicode_AS_UNICODE(po), flag, mode);
                Py_END_ALLOW_THREADS
                if (fd < 0)
                    return posix_error();
                return PyInt_FromLong((long)fd);
            }
            /* Drop the argument parsing error as narrow strings
               are also valid. */
            PyErr_Clear();
        }
    #endif

        if (!PyArg_ParseTuple(args, "eti|i",
                              Py_FileSystemDefaultEncoding, &file,
                              &flag, &mode))
            return NULL;

        Py_BEGIN_ALLOW_THREADS
        fd = open(file, flag, mode);
        Py_END_ALLOW_THREADS
        if (fd < 0)
            return posix_error_with_allocated_filename(file);
        PyMem_Free(file);
        return PyInt_FromLong((long)fd);
    }

前半部分代码是windows用的，linux的在后半部。先获得参数: file, flag,
可选的mode。然后调用open系统函数，最后返回一个Int类型的python对象。

仔细观察，如果参数有错误，返回NULL，在python层面则表现为抛出了异常，
由此是否可以猜测，对于此函数来说，返回值为NULL就表示有异常？还有什么要注意的吗？

再看，如果是文件不存在，open失败，同样在上层表现为异常，但是返回前的处理却不一样::

    static PyObject *
    posix_error_with_allocated_filename(char* name)
    {
        PyObject *rc = PyErr_SetFromErrnoWithFilename(PyExc_OSError, name);
        PyMem_Free(name);
        return rc;
    }

可以看出，open之前，file还是一个空指针，没有指向分配的内存，所以只返回NULL就足够了。
open之后，不管是成功还是失败，file指针都需要被释放掉。这是需要特别小心的地方，一旦
处理不到，就会造成内存泄露。原则是，在返回之前，一定要把已申请的资源处理好。

现在有了足够的信心，照着原有代码的例子，我们可以抛出自己的异常。用什么函数呢？
PyErr_SetFromErrnoWithFilename 看着像和异常有关，翻看代码，可以看到类似函数::

    +2282
    if (len >= MAX_PATH) {
        PyErr_SetString(PyExc_ValueError, "path too long");
        return NULL;
    }

    +2831
    else if (!PyTuple_Check(arg) || PyTuple_Size(arg) != 2) {
        PyErr_SetString(PyExc_TypeError,
                        "utime() arg 2 must be a tuple (atime, mtime)");
        goto done;
    }
 
PyErr_SetString 抛出一个纯c字符串，不需要担心对象引用，正是我们想要的。第一个
参数为异常的类型。

file是 `char *` 类型，这意味是我们可以用strcmp。

代码如下::

    jaime@ideer:~/source/Python-2.6.7$ git df
    diff --git a/Modules/posixmodule.c b/Modules/posixmodule.c
    index 822bc11..7501f0d 100644
    --- a/Modules/posixmodule.c
    +++ b/Modules/posixmodule.c
    @@ -6337,11 +6337,19 @@ posix_open(PyObject *self, PyObject *args)
         }
     #endif
     
    +    printf("Entering posix_open\n");
    +
         if (!PyArg_ParseTuple(args, "eti|i",
                               Py_FileSystemDefaultEncoding, &file,
                               &flag, &mode))
             return NULL;
     
    +    if (strcmp(file, "hello") == 0) {
    +        PyErr_SetString(PyExc_ValueError, "Hello, exception!");
    +        PyMem_Free(file);
    +        return NULL;
    +    }
    +
         Py_BEGIN_ALLOW_THREADS
         fd = open(file, flag, mode);
         Py_END_ALLOW_THREADS
    jaime@ideer:~/source/Python-2.6.7$


输出::

    jaime@ideer:~/source/Python-2.6.7$ ./python 
    Python 2.6.7 (r267:88850, Sep 10 2011, 12:12:00) 
    [GCC 4.5.2] on linux2
    Type "help", "copyright", "credits" or "license" for more information.
    >>> import os
    >>> os.open()
    Entering posix_open
    Traceback (most recent call last):
      File "<stdin>", line 1, in <module>
    TypeError: function takes at least 2 arguments (0 given)
    >>> os.open('hello', os.O_RDONLY)
    Entering posix_open
    Traceback (most recent call last):
      File "<stdin>", line 1, in <module>
    ValueError: Hello, exception!
    >>> os.open('test', os.O_RDONLY)
    Entering posix_open
    Traceback (most recent call last):
      File "<stdin>", line 1, in <module>
    OSError: [Errno 2] No such file or directory: 'test'
    >>> os.open('test', os.O_WRONLY | os.O_CREAT)
    Entering posix_open
    3
    >>> 

注意三个异常发生的时刻，以及类型TypeError, ValueError,
OSError。另一个有趣的函数是 PyErr_Format，可以抛出一个格式化的字符串。

Python/builtinmodule.c +188::

    if (kwdict != NULL && !PyDict_Check(kwdict)) {
        PyErr_Format(PyExc_TypeError,
                     "apply() arg 3 expected dictionary, found %s",
                     kwdict->ob_type->tp_name);
        goto finally;
    }
 
更多异常处理函数参见 Include/pyerrors.h, Python/errors.c。

PyArg_ParseTuple 参见 The Python/C API。


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

所以os.open实际上是posix.open，代码在Modules/posixmodule.c posix_open。

::
    >>> import os
    >>> import posix
    >>> id(os.open)
    3077348460L
    >>> id(posix.open)
    3077348460L
    >>>

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



