---
title: Python 中的下划线
copyright: true
permalink: 1
top: 0
date: 2019-05-07 11:30:08
tags: python
categories: python
password:
---

python 作为动态语言，不同于java通过关键字修饰变量作用域，Python中，解释器通过读取单下划线和双下划线来实现不同的含义。<!--more-->


python中，通过定义以下五种下划线模式和命名约定：

单前导下划线：_var

单末尾下划线：var_

双前导下划线：__var

双前导和末尾下划线：__ var__

单下划线：_

### 单前导下划线：_var

python中的单前导下划线在python中有一个约定俗成的含义，当它定义一个变量或则函数名时，表明该的变量或函数仅供内部使用。但这不是强制规定，外部依旧可以访问。

```python
# test1.py:

def a():
   return 23

def _b():
   return 42

```
外部调用通过通配符的方式调用不到单前导下划线定义的变量或函数，常规导入不受前导单个下划线命名约定的影响
```python

>>> from my_module import *
>>> a()
23
>>> _b()
NameError: "name '_b' is not defined"
>>> from my_module import _b
>>> _b()
24
```

### 单末尾下划线 var_

单下划线结尾的变量：用于避免于Python关键字冲突的变量，如class_：

Tkinter.Toplevel(master, class_='ClassName')

### 双前导下划线 __var

双下划线开头的变量：它在模块中还是当作单下划线看待，但出现在类中作为类属性就不一样了，不能直接访问.
```python
In [3]: class A(object):
   ...:     a = 40
   ...:     _a = 50
   ...:     __a = 60  # _Foo__boo 
   ...:     def __init__(self):
   ...:         self.__b = 70
   ...:     def __test(self):
   ...:         print("__test")
```
单前导下划线标识的类变量依旧可以被调用，但双前导下划线标识的变量或函数会被解释器隐藏
```python
In [4]: A._a
Out[4]: 50

In [5]: A.__a
---------------------------------------------------------------------------
AttributeError                            Traceback (most recent call last)
<ipython-input-9-aca55768bfd2> in <module>
----> 1 A.__a

AttributeError: type object 'A' has no attribute '__a'

```

通过__dict__可以查看对象属性，通过A._A__a可访问__a。
```python
In [6]: A.__dict__
Out[6]: 
mappingproxy({'__module__': '__main__',
              'a': 40,
              '_a': 50,
              '_A__a': 60,
              '__init__': <function __main__.A.__init__(self)>,
              '_A__test': <function __main__.A.__test(self)>,
              '__dict__': <attribute '__dict__' of 'A' objects>,
              '__weakref__': <attribute '__weakref__' of 'A' objects>,
              '__doc__': None})
              
In [7]: A._A__a
Out[7]: 60
```
实例化对象也是通过加前单下划线类名访问变量或函数

```python
In [8]: a_obj._A__b
Out[8]: 70

In [9]: a_obj._A__test()
__test

```
### 双前导和双末尾下划线 __ var__

Python保留了有双前导和双末尾下划线的名称，用于特殊用途。 

__ init__对象构造函数

__ call__它使得一个对象可以被调用

### 单下划线 _

当赋值的变量无关紧要时，可以通过"_"表示其为一个临时值
```python
In [26]: color,_,_,_ = ('red', 'auto', 12, 3812.4)

In [28]: for key,_ in {"a":"A","b":"B"}.items():
    ...:     print(key)
    ...: 
a
b
```

"_"是大多数Python REPL中的一个特殊变量，它表示由解释器评估的最近一个表达式的结果。
```python
5+6
Out[2]: 11
_
Out[3]: 11
list()
Out[4]: []
_.append(1)
_
Out[6]: [1]
```

### ALL
看Django源码的时候会看到在代码最前面会有一个__ all__，其实__all__对象是装有字符串的列表对象，他会覆盖 from import * 的默认行为，可以把下划线开头的变量的字符串形式加入到__all__中，这样 import * 也能看到这些变量。 


参考

[https://dbader.org/blog/meaning-of-underscores-in-python](https://dbader.org/blog/meaning-of-underscores-in-python)

[https://zhuanlan.zhihu.com/p/36173202](https://zhuanlan.zhihu.com/p/36173202)

[https://foofish.net/python_xiahuaxian.html](https://foofish.net/python_xiahuaxian.html)