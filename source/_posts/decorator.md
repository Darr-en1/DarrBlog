---
title: decorator
copyright: true
permalink: 1
top: 0
date: 2019-03-12 20:33:10
tags:
    - python
    - 设计模式
    - 装饰器
categories: python
password:
---

python中，函数可以像变量一样当作参数传递给另一个函数， 装饰器则沿用了这一特性，在不改变既有代码的前提下，增加功能。<!--more-->


### 描述
装饰器本质上是一个 Python 函数或类，返回值也是一个函数/类对象。

**原则**：
1. 不修改被修饰函数的源代码
2. 不修改被修饰函数的调用方式

装饰器 = 高阶函数+函数嵌套+闭包


#### 高阶函数
函数接受参数为函数名或函数返回值是函数名

```python
def add(x, y, f):
    return f(x) + f(y)
    
add(-5,6,abs) # 传入两个变量和一个绝对值函数名   return 11
```


#### 函数嵌套
函数内部定义函数

```python
def outer():
    def inner():
        print("inner")
    print("outer")
    inner()
    
outer()
```

#### 闭包

闭包，顾名思义，就是一个封闭的包裹，里面包裹着自由变量，就像在类里面定义的属性值一样，自由变量的可见范围随同包裹，哪里可以访问到这个包裹，哪里就可以访问到这个自由变量。

通常函数被调用后，局部变量变失去作用域，闭包可以避免使用全局变量，使得变量脱离了函数本身的作用范围，局部变量依旧可以被访问得到

```python
def adder(x):
    def wrapper(y):
        return x + y
    return wrapper
    
adder5 = adder(5)
# 输出 15
adder5(10)
# 输出 11
adder5(6)
```



### 简单装饰器

函数进入和退出时 ，被称为一个横切面，这种编程方式被称为面向切面的编程。在java    Spring框架aop就是面向切面编程

```python
import time

def timmer(func):
    def wrapper(*args,**kwargs):
        start_time=time.time()
        res=func(*args,**kwargs)
        stop_time = time.time()
        print(f'运行时长{stop_time-start_time}')
        return res
    return wrapper

def IO_operation():
    time.sleep(4)

IO_operation = timmer(IO_operation)  # 因为装饰器 timmer(IO_operation) 返回是函数对象 wrapper，这条语句相当于  IO_operation = wrapper
IO_operation()
```


#### **@ 语法糖** 简化调用

```python
@timmer
def IO_operation():
    time.sleep(4)

IO_operation()
```

### 带参数的装饰器
装饰器的语法允许我们在调用时，提供其它参数,它实际上是对原有装饰器的一个函数封装，并返回一个装饰器.


```python
def timmer(level):
    def decorator(func):
        def wrapper(*args,**kwargs):
            start_time=time.time()
            res=func(*args,**kwargs)
            stop_time = time.time()
            if level=1:
            print(f'1:{stop_time-start_time}')
            if level=2:
            print(f'2:{stop_time-start_time}')
            return res
        return wrapper
    return decorator
    
 def IO_operation():
    time.sleep(4)
 
    
@timmer(1)
def IO_operation():
    time.sleep(4)

IO_operation()
```
### 类装饰器
类装饰器主要依靠类的__call__方法,当一个类如果实现了__call__方法，那么该类的实例对象的行为就是一个函数，是一个可以被调用（callable)的对象。
```python
class Add:
    def __init__(self, n):
         self.n = n
    def __call__(self, x):
        return self.n + x

>>> add = Add(1)
>>> add(4)
>>> 5
```
确定对象是否为可调用对象通过内置函数callable来判断
```python
>>> callable(foo)
True
>>> callable(1)
False
>>> callable(int)
True
```

#### 类装饰器实现
```python

class Foo(object):
    def __init__(self, func):
        self._func = func

    def __call__(self):
        print ('class decorator runing')
        self._func()
        print ('class decorator ending')

@Foo
def bar():
    print ('bar')

bar()
```
### functools.wraps保留函数元信息

使用装饰器极大地复用了代码，但是他有一个缺点就是原函数的元信息被内部函数得元信息替代，python内部提供functools.wraps装饰器，它能把原函数的元信息拷贝到装饰器里面的 func 函数中，这使得装饰器里面的 func 函数也有和原函数 foo 一样的元信息了。


```python
from functools import wraps
def logged(func):
    @wraps(func)
    def with_logging(*args, **kwargs):
        print func.__name__      # 输出 'f'
        print func.__doc__       # 输出 'does some math'
        return func(*args, **kwargs)
    return with_logging

@logged
def f(x):
   """does some math"""
   return x + x * x
```

### 装饰器顺序
一个函数还可以同时定义多个装饰器，比如：
```python
@a
@b
@c
def f ():
    pass
    
>>>f = a(b(c(f)))  #执行顺序是从里到外
```



参考：

[https://foofish.net/python-closure.html](https://foofish.net/python-closure.html)

[https://foofish.net/python-decorator.html](https://foofish.net/python-decorator.html)

[https://foofish.net/function-is-first-class-object.html](https://foofish.net/function-is-first-class-object.html)
