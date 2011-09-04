
sys.path[0]是空字符串，怎么回事？

交互模式下，这样设置表示当前目录

如果你把.py脚本文件作为参数传递给python解释器，那么
sys.path[0]通常将是该文件所在目录，即os.path.dirname()



PYTHONHOME
PYTHONPATH
prefix

多版本python
python在启动的时候，会根据PYTHONHOME查看自身bin所在位置，
从而推断出相应版本的标准lib所在位置。

python运行需要的信息如下：
python      可执行文件
系统标准lib 用.py写的自带模块，.so扩展
用户模块    用户编写的.py文件
第三方包 你的程序中导入的第三方模块  

知道了以上信息，就可以构建一个完整的python运行环境了。


sys.executable

Get_Path函数



