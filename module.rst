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
从sock.connect开始，你将要潜入C代码，在 Modules/socketmodule.c +2027::

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

`wsgi <http://www.python.org/dev/peps/pep-0333/>`_ 规定了两个函数, write 
和start_response::

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



