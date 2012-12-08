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

我们知道python模块都是从本地文件系统加载进来的，那么这个最底层的从磁盘读取文件的地方在哪里？

- 读取 .py
- 生成 .pyc，写入磁盘
- 执行编译过的python VM指令.pyc

模块来源也可能是从网络读取，或者程序内生成。假设有一个data数据块，能不能将它变成python模块呢？

如果data是合法的py程序，则可以使用eval，compile等，将其编译为一个module。

如果data是其他语言的程序，比如一个c extension的代码，能否在python内将其编译为python模块呢？

如果data是一个精巧构造的二进制块，能否让python vm将其认为是一个python codeobject呢？
用纯粹的py程序，能否构造一个codeobject？在dynload, .so的加载被禁用的情况下，用纯粹的py能否实现动态加载.so的功能？

可以认为是不可能的，dl只有os配合才可以。或者只有构造缓冲区溢出，让程序的pc执行到你构造的代码区。

tracer.runfunc
---------------------------

PyEval_EvalCode
    PyEval_EvalCodeEx
        PyEval_EvalFrameEx

Python的线程：EvalFrameEx函数同时存在多个指令流，每个执行实例对应于一个native的c thread

Python/pythonrun.c 编译并执行一个文件的入口::

    PyObject *
    PyRun_FileExFlags(FILE *fp, const char *filename, int start, PyObject *globals,
                      PyObject *locals, int closeit, PyCompilerFlags *flags)
    {
        PyObject *ret;
        mod_ty mod;
        PyArena *arena = PyArena_New();
        if (arena == NULL)
            return NULL;

        mod = PyParser_ASTFromFile(fp, filename, start, 0, 0,
                                   flags, NULL, arena);
        if (closeit)
            fclose(fp);
        if (mod == NULL) {
            PyArena_Free(arena);
            return NULL;
        }
        ret = run_mod(mod, filename, globals, locals, flags, arena);
        PyArena_Free(arena);
        return ret;
    }

    static PyObject *
    run_mod(mod_ty mod, const char *filename, PyObject *globals, PyObject *locals,
             PyCompilerFlags *flags, PyArena *arena)
    {
        PyCodeObject *co;
        PyObject *v;
        co = PyAST_Compile(mod, filename, flags, arena);
        if (co == NULL)
            return NULL;
        v = PyEval_EvalCode(co, globals, locals);
        Py_DECREF(co);
        return v;
    }

Modules/main.c Py_Main 分析命令行参数，初始化环境，启动解释器

freevars
cellvars

fastlocals
freevars
consts

c_tracefunc
c_profilefunc



mainloop:

    continue  继续下一条指令, 对应于for, 不对本条指令检测错误
    break 对应于switch
    goto fast_next_op 快速执行到下一条指令, 没有错误，不用对本条指令进行错误检测，同时跳过下一条指令的tsc，线程切换ticker 代码, tsc时间统计？

 
 
EvalFrameEx:

    init_frame

    for(;;):
        init_op
        fast_next_op:
            init_op_fast

        switch(op):
            ...

        on_error: 在执行每条指令后，检测是否有错误发生

check frame error


 有些指令并没有清空堆栈，由vm负责执行：
 
     case UNARY_POSITIVE:
            v = TOP();
            x = PyNumber_Positive(v);
            Py_DECREF(v);
            SET_TOP(x);
            if (x != NULL) continue;
            break;
留了一个NULL在堆栈顶


    assert(why != WHY_YIELD);
    /* Pop remaining stack entries. */
    while (!EMPTY()) {
        v = POP();
        Py_XDECREF(v);
    }


Object/abstract.c is interesting

PyEval_CallObject 执行object的tp_call


RETURN_VALUE 函数调用返回,仍在当前frame

YIELD_VALUE 跳出frame


exec_statement VM自身的递归

build_class 生成一个class object: metaclass, bases问题

访问属性:
PyObject_SetAttr
PyObject_GetAttr 



名字解析, var, identitifer::

先查找f_locals, 看是dict还是object，然后查找f_globals, 最后f_builtins

     case LOAD_NAME:
                w = GETITEM(names, oparg);
                if ((v = f->f_locals) == NULL) {
                    PyErr_Format(PyExc_SystemError,
                                 "no locals when loading %s",
                                 PyObject_REPR(w));
                    why = WHY_EXCEPTION;
                    break;
                }
                if (PyDict_CheckExact(v)) {
                    x = PyDict_GetItem(v, w);
                    Py_XINCREF(x);
                }
                else {
                    x = PyObject_GetItem(v, w);
                    if (x == NULL && PyErr_Occurred()) {
                        if (!PyErr_ExceptionMatches(
                                        PyExc_KeyError))
                            break;
                        PyErr_Clear();
                    }
                }
                if (x == NULL) {
                    x = PyDict_GetItem(f->f_globals, w);
                    if (x == NULL) {
                        x = PyDict_GetItem(f->f_builtins, w);
                        if (x == NULL) {
                            format_exc_check_arg(
                                        PyExc_NameError,
                                        NAME_ERROR_MSG, w);
                            break;
                        }
                    }
                    Py_INCREF(x);
                }
                PUSH(x);
                continue;

LOAD系的指令:             
load_attr 获取属性
load_name 加载name对应(binding)的object到堆栈上
load_const
load_fast
store_attr


fast_block_end: 异常处理机制

jump_if_true 如果栈顶为true则跳转，否则要pop栈顶继续顺序执行。为什么需要一个单独的
pop_top指令呢?

因为TOS可能会被多条指令共享，不用每次push，pop提高效率，如::

    from setuptools import setup, find_packages


  1           0 LOAD_CONST               0 (-1)
              3 LOAD_CONST               1 (('setup', 'find_packages'))
              6 IMPORT_NAME              0 (setuptools)
              9 IMPORT_FROM              1 (setup)
             12 STORE_NAME               1 (setup)
             15 IMPORT_FROM              2 (find_packages)
             18 STORE_NAME               2 (find_packages)
             21 POP_TOP  

从编译后的指令可以看出，setuptools这个模块一直唯一栈中，被后续的两条IMPORT_FROM引用，用完之后再被显式pop掉,
很自然的使用方式。

三条导入模块指令:
import_name  import语句
import_from  从模块中导入部分名字
import_star  导入所有名字

从当前frame的builtins获得__import__，调用该函数完成真正的导入操作::

 case IMPORT_NAME:
            w = GETITEM(names, oparg);
            x = PyDict_GetItemString(f->f_builtins, "__import__");
            if (x == NULL) {
                PyErr_SetString(PyExc_ImportError,
                                "__import__ not found");
                break;
            }
            Py_INCREF(x);
            v = POP();
            u = TOP();
            if (PyInt_AsLong(u) != -1 || PyErr_Occurred())
                w = PyTuple_Pack(5,
                            w,
                            f->f_globals,
                            f->f_locals == NULL ?
                                  Py_None : f->f_locals,
                            v,
                            u);
            else
            ....
        
http://docs.python.org/library/functions.html#__import__

FOR_ITER 读取iter当前的值，iter的状态在对象内部维护，是循环的开始，在下一个循环仍会跳到该指令。如果iter耗尽，则结束迭代。 :: 

    21:32 jaime@oldtown source$ python -m dis c.py 
      1           0 LOAD_CONST               0 (1)
                  3 STORE_NAME               0 (a)

      3           6 SETUP_LOOP              25 (to 34)
                  9 LOAD_NAME                1 (range)
                 12 LOAD_CONST               1 (3)
                 15 CALL_FUNCTION            1
                 18 GET_ITER            
            >>   19 FOR_ITER                11 (to 33)
                 22 STORE_NAME               2 (i)

      4          25 LOAD_NAME                2 (i)
                 28 PRINT_ITEM          
                 29 PRINT_NEWLINE       
                 30 JUMP_ABSOLUTE           19
            >>   33 POP_BLOCK           

      6     >>   34 LOAD_CONST               2 (2)
                 37 STORE_NAME               0 (a)
                 40 LOAD_CONST               3 (None)
                 43 RETURN_VALUE        
    21:32 jaime@oldtown source$ cat c.py 
    a = 1

    for i in range(3):
        print i

    a = 2
    21:32 jaime@oldtown source$ 


    case FOR_ITER:
        /* before: [iter]; after: [iter, iter()] *or* [] */
        v = TOP();
        x = (*v->ob_type->tp_iternext)(v); // 调用iter的next方法，怎么关联到自定义的__next__方法，
        // Tools/framer/framer/slots.py?
        if (x != NULL) {
            PUSH(x);
            PREDICT(STORE_FAST);
            PREDICT(UNPACK_SEQUENCE);
            continue; // Normal case
        }
        if (PyErr_Occurred()) {
            if (!PyErr_ExceptionMatches(
                            PyExc_StopIteration))
                break; // 不是StopIteration，出错了，跳转到指令错误处理代码
            PyErr_Clear();
        }
        /* iterator ended normally */
        x = v = POP();
        Py_DECREF(v);
        JUMPBY(oparg);
        continue;


for, while, try/except/finally，创建一个新的block::

        case SETUP_LOOP:
        case SETUP_EXCEPT:
        case SETUP_FINALLY:
            /* NOTE: If you add any new block-setup opcodes that
               are not try/except/finally handlers, you may need
               to update the PyGen_NeedsFinalizing() function.
               */

            PyFrame_BlockSetup(f, opcode, INSTR_OFFSET() + oparg,
                               STACK_LEVEL());
            continue;

Frame & Block WTF?


build_class, make_function指令多次执行的问题？生成多个object？

function系指令::

CALL_FUNCTION
MAKE_FUNCTION
MAKE_CLOSURE

do_call, function_function: Recursive VM 函数调用，实际上是递归调用PyEval_EvalFrameEx

::

    static PyObject *
    fast_function(PyObject *func, PyObject ***pp_stack, int n, int na, int nk)
    {
        PyCodeObject *co = (PyCodeObject *)PyFunction_GET_CODE(func);
        PyObject *globals = PyFunction_GET_GLOBALS(func);
        PyObject *argdefs = PyFunction_GET_DEFAULTS(func);
        PyObject **d = NULL;
        int nd = 0;

        PCALL(PCALL_FUNCTION);
        PCALL(PCALL_FAST_FUNCTION);
        if (argdefs == NULL && co->co_argcount == n && nk==0 &&
            co->co_flags == (CO_OPTIMIZED | CO_NEWLOCALS | CO_NOFREE)) {
            PyFrameObject *f;
            PyObject *retval = NULL;
            PyThreadState *tstate = PyThreadState_GET();
            PyObject **fastlocals, **stack;
            int i;

            PCALL(PCALL_FASTER_FUNCTION);
            assert(globals != NULL);
            /* XXX Perhaps we should create a specialized
               PyFrame_New() that doesn't take locals, but does
               take builtins without sanity checking them.
            */
            assert(tstate != NULL);
            // 每次调用都生成新的frame
            f = PyFrame_New(tstate, co, globals, NULL);
            if (f == NULL)
                return NULL;

            fastlocals = f->f_localsplus;
            stack = (*pp_stack) - n;

            for (i = 0; i < n; i++) {
                Py_INCREF(*stack);
                fastlocals[i] = *stack++;
            }
            retval = PyEval_EvalFrameEx(f,0);
            ++tstate->recursion_depth;
            Py_DECREF(f);
            --tstate->recursion_depth;
            return retval;
        }
        if (argdefs != NULL) {
            d = &PyTuple_GET_ITEM(argdefs, 0);
            nd = Py_SIZE(argdefs);
        }
        return PyEval_EvalCodeEx(co, globals,
                                 (PyObject *)NULL, (*pp_stack)-n, na,
                                 (*pp_stack)-2*nk, nk, d, nd,
                                 PyFunction_GET_CLOSURE(func));
    }


call_function, ext_do_call: 函数调用入口

    static PyObject *
    call_function(PyObject ***pp_stack, int oparg
    #ifdef WITH_TSC
                    , uint64* pintr0, uint64* pintr1
    #endif
                    )
    {
        int na = oparg & 0xff;
        int nk = (oparg>>8) & 0xff;
        int n = na + 2 * nk;
        PyObject **pfunc = (*pp_stack) - n - 1;
        PyObject *func = *pfunc;
        PyObject *x, *w;

        /* Always dispatch PyCFunction first, because these are
           presumed to be the most frequent callable object.
        */
        if (PyCFunction_Check(func) && nk == 0) {
            int flags = PyCFunction_GET_FLAGS(func);
            PyThreadState *tstate = PyThreadState_GET();

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
            else {
                PyObject *callargs;
                callargs = load_args(pp_stack, na);
                READ_TIMESTAMP(*pintr0);
                C_TRACE(x, PyCFunction_Call(func,callargs,NULL));
                READ_TIMESTAMP(*pintr1);
                Py_XDECREF(callargs);
            }
        } else {
            if (PyMethod_Check(func) && PyMethod_GET_SELF(func) != NULL) {
                /* optimize access to bound methods */
                PyObject *self = PyMethod_GET_SELF(func);
                PCALL(PCALL_METHOD);
                PCALL(PCALL_BOUND_METHOD);
                Py_INCREF(self);
                func = PyMethod_GET_FUNCTION(func);
                Py_INCREF(func);
                Py_DECREF(*pfunc);
                *pfunc = self;
                na++;
                n++;
            } else
                Py_INCREF(func);
            READ_TIMESTAMP(*pintr0);
            if (PyFunction_Check(func))
                x = fast_function(func, pp_stack, n, na, nk);
            else
                x = do_call(func, pp_stack, na, nk);
            READ_TIMESTAMP(*pintr1);
            Py_DECREF(func);
        }

        /* Clear the stack of the function object.  Also removes
           the arguments in case they weren't consumed already
           (fast_function() and err_args() leave them on the stack).
         */
        while ((*pp_stack) > pfunc) {
            w = EXT_POP(*pp_stack);
            Py_DECREF(w);
            PCALL(PCALL_POP);
        }
        return x;
    }

frameobject, codeobject, blockobject???

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
