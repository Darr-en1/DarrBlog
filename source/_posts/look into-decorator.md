---
title: 深入理解python装饰器
copyright: true
permalink: 1
top: 0
date: 2020-01-16 16:22:49
tags: 
    - 装饰器
    - python
categories: python
password:
---
装饰器在python中应用非常广泛，这种模式允许向一个现有的对象添加新的功能，同时又不改变其结构。
一直以来我都装饰器的特性是可以将被装饰的函数替换成其他函数，实现功能的扩充。
但最近阅读<<流畅的python>>的时候，才意识到其另一大特性是<font color=#00ffff>装饰器在加载模块是会立即执行。</font> <!--more-->

### 演示

通常，模块被加载时，模块内的装饰器就会被执行，和类变量的被执行方式一样。
凭借这个特性，下面的结果就显得合理：

```python
func_list = []


def promotion(func):
    func_list.append(func)
    return func


@promotion
def add(x, y):
    return x + y


@promotion
def sub(x, y):
    return x - y


@promotion
def mul(x, y):
    return x * y


@promotion
def div(x, y):
    return x / y


def output(x, y):
    return {func.__name__: func(x, y) for func in func_list}


if __name__ == '__main__':
    print(output(10, 2))
```
output:

```python
{'add': 12, 'sub': 8, 'mul': 20, 'div': 5.0}
```

### 扩展

如果有参与过异步框架下的开发工作，肯定会出现底层未支持或无现成可用的异步package时，
就会通过 run_in_executor让当前阻塞方式异步执行来避免异步框架因为局部阻塞导致并发严重下降。
每一个方法都的重复此操作，未免太繁琐，装饰器就很好的可以解决这个问题
```python
def coroutine_executor(func):
    @wraps(func)
    def wrapper(*args, **kwargs):
        return asyncio.get_event_loop().run_in_executor(ThreadPoolExecutor(), partial(func, *args, **kwargs))

    return wrapper
```
想要开始使用协程的时候必须通过next(...)方式激活协程，如果不预激，这个协程就无法使用，这种重复繁琐的事情交给装饰器再合适不过了
```python
def coroutine(func):
    @wraps(func)
    def primer(*args,**kwargs):
        gen = func(*args,**kwargs)
        next(gen)
        return gen
    return primer
```
倘若有些方法执行耗时未知，但又不想改变其同步的运行模式，理论上在一定时间范围内可以接受。
这时候，可以通过ThreadPoolExecutor和装饰器来实现这个功能。
```python
def async_timeout(timeout=20):
    def decorate(func):
        @wraps(func)
        def wrapper(*args, **kwargs):
            return ThreadPoolExecutor().submit(partial(func, *args, **kwargs)).result(timeout=timeout)

        return wrapper

    return decorate
```
最近发现一个开源的package [timeout-decorator][https://github.com/pnpnpn/timeout-decorator],
浏览了一下源码，就一个文件，写的非常详尽可以浏览使用。但年代感沉重，但还是挺有借鉴价值的。
以上就是在我开发过程中通过装饰器带来的便利。

[https://github.com/pnpnpn/timeout-decorator]: https://github.com/pnpnpn/timeout-decorator