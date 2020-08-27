---
title: gevent魔法和test中的应用实践
copyright: true
permalink: 1
top: 0
date: 2020-08-26 14:38:59
tags:
    - python
    - test
    - 源码
categories: python
password:
---

了解异步编程的同学对于gevent一定不会陌生,但大部分人对于gevent可以在不修改任何代码的前提下将原始同步代码更替成异步代码的原理知之甚少。
很多人很胆怯阅读源码,但通过阅读源码，你既能学习到正确的编码规范，也能学习到好的编程思路。
接下来我带领大家一起来揭开gevent这层神秘面纱，及借助gevent思想在项目test中的应用。<!--more-->

### gevent猴子补丁

gevent的用途是在不侵入原始代码的前提下，让你可以很方便的导入非阻塞的模块，无需特意引入。
其实现原理是将原始库的sync func替换成gevent内置的 async func。
#### 模块补丁实现

**暴力替换**:在python sys变量中将目标模块直接替换成新的模块
```python
# module_1.py
def print1():
    print("print 1")
```

```python
# module_2.py
def print2():
    print("print 2")
```
```python
# test.py
import sys

import module_1

module_1.pprint()
del sys.modules['module_1']
sys.modules['module_1'] = __import__('module_2')
import module_1

module_1.pprint()
```
output:
```text
print 1
print 2
```


**非暴力替换**:将目标模块特定func 替换成新模块的特定func

```python
import module_1
import module_2


def monkey_patch():
    module_1.pprint = module_2.pprint


monkey_patch()
module_1.pprint()
```

output:
```text
print 2
```

从上面两个例子中你可以很清晰的看到虽然调用的是目标函数,但实际上执行的确实被替换的新的函数。

#### gevent monkey.parch_all源码解析
patch_all 就是用gevent的async的func替换python 原生库 sync的func
patch_all针对模块的引入是有顺序的, 因为他们之间有依赖的关系。

```python
def patch_all(socket=True, dns=True, time=True, select=True, thread=True, os=True, ssl=True,
              httplib=False, # Deprecated, to be removed.
              subprocess=True, sys=False, aggressive=True, Event=True,
              builtins=True, signal=True,
              queue=True,
              **kwargs):
    # locals() 函数会以字典类型返回当前位置的全部局部变量。
    _warnings, first_time, modules_to_patch = _check_repatching(**locals())
    # 存在 warning 并且不是第一次加载 return
    if not _warnings and not first_time:
        return

    # 省略 ...
    
    # 内置对于python原生库的封装，进行sync向aysnc的替换操作
    if os:
        patch_os()
    if time:
        patch_time()
    # 省略 ...

```

_check_repatching对patch_all 函数局部变量校验:
- 是否第一次加载
- 变量参数书否一一致

```python
saved = {}
def _check_repatching(**module_settings):
    _warnings = []
    key = '_gevent_saved_patch_all'
    del module_settings['kwargs']
    # 前后加载的参数不一致 warning
    if saved.get(key, module_settings) != module_settings:
        _queue_warning("Patching more than once will result in the union of all True"
                       " parameters being patched",
                       _warnings)
                       
    first_time = key not in saved
    saved[key] = module_settings
    return _warnings, first_time, module_settings
```

_patch_module获取gevent内置的module和目标module 

```python
@_ignores_DoNotPatch
def patch_os():
    _patch_module('os')
    
    
def _patch_module(name, items=None, _warnings=None, _notify_did_subscribers=True):

gevent_module = getattr(__import__('gevent.' + name), name)
module_name = getattr(gevent_module, '__target__', name)
target_module = __import__(module_name)

patch_module(target_module, gevent_module, items=items,
             _warnings=_warnings,
             _notify_did_subscribers=_notify_did_subscribers)

return gevent_module, target_module
```

patch_module 获取target_module需要被替换的attr list 可以通过 items 指定，
也可以在source_module的变量（__ _implements_ __）中写入。然后调用patch_item进行
函数替换操作。

```python

_NONE = object()

def patch_module(target_module, source_module, items=None,
                 _warnings=None,
                 _notify_did_subscribers=True):
                 
    if items is None:
        items = getattr(source_module, '__implements__', None)
        if items is None:
            raise AttributeError('%r does not have __implements__' % source_module)
            
    for attr in items:
        patch_item(target_module, attr, getattr(source_module, attr))


def patch_item(module, attr, newitem):
    olditem = getattr(module, attr, _NONE)
    if olditem is not _NONE:
        saved.setdefault(module.__name__, {}).setdefault(attr, olditem)
    setattr(module, attr, newitem)

```
以上就是gevent 通过 内置的async func 替换 python 内置target sync func 的基本流程，是不是也并不难理解。

### test中的启发

工作中,为了保证代码质量，避免出现一想不到的错误，编写测试用例是在开发过程中视为不可忽视的一环，甚至还出现了TDD等开发流程。

测试一般都是本着不侵入目标代码的前提下，对目标代码功能进行测试。

但很多时候不避免的就会需要在目标代码上做一些微调来适配测试。
如:项目中使用了request,但是在测试环境下，不需要真正的请求发送，可以参考[httmock](https://github.com/patrys/httmock)。

倘若要满足不侵入的前提条件，就需要mock出一个服务。在测试阶段使用mock的服务。

本人从事的项目还参与硬件的交互。项目中内置了一个模块封装与设备的交互。上层业务代码只需要调用模块的func即可。
但是在test时，存在一个难点: 需要有已经连接的设备。

怎么剔除这个依赖呢?

#### 装饰器替换 

将装饰器作用于被调用模块的class或则func，替换目标函数

简单demo:
```python
def pprint(x):
    print(222, x)


def deco(item):
    def inner():
        setattr(item, "pprint", pprint)
        return item

    return inner


@deco
class TT:
    def pprint(self, x):
        print(111, x)


TT().pprint(233)

```

output:
```text
222 233
```

缺点：
- 存在侵入代码的情况
- 对于源码库或则第三方代码并不能直接加装饰器，需要做一层封装后才能实现


#### gevent方案进行替换


如下目录结构
```text
├── app
│   └── libs
│       └── http_client.py
└── mock
    └── monkey
    └──  http_client.py
└── test.py
```

模块替换代码
```python
# mock.monkey.py

import importlib

saved = {}

_NONE = object()


def patch_all(httpclint=True, **kwargs):
    first_time, modules_to_patch = _check_repatching(**locals())
    if not first_time:
        return
    if httpclint:
        patch_httpclint()


def _check_repatching(**module_settings):
    key = '_gevent_saved_patch_all'
    del module_settings['kwargs']

    first_time = key not in saved
    saved[key] = module_settings
    return first_time, module_settings


def patch_httpclint():
    _patch_module('http_client')


def _patch_module(name, items=None, _warnings=None, _notify_did_subscribers=True):
    gevent_module = importlib.import_module('mock.' + name)
    module_name = getattr(gevent_module, '__target__', name)
    target_module = importlib.import_module(module_name)

    patch_module(target_module, gevent_module, items=items,
                 _warnings=_warnings,
                 _notify_did_subscribers=_notify_did_subscribers)

    return gevent_module, target_module


def patch_module(target_module, source_module, items=None,
                 _warnings=None,
                 _notify_did_subscribers=True):

    if items is None:
        items = getattr(source_module, '__implements__', None)
        if items is None:
            raise AttributeError('%r does not have __implements__' % source_module)

    for attr in items:
        patch_item(target_module, attr, getattr(source_module, attr))

    return True


def patch_item(module, attr, newitem):
    olditem = getattr(module, attr, _NONE)
    if olditem is not _NONE:
        saved.setdefault(module.__name__, {}).setdefault(attr, olditem)
    setattr(module, attr, newitem)

```

```python
# mock.monkey.py

# 需要被替换的模块的位置
__target__ = "app.libs.http_client"

# 被替换模块的方法
__implements__ = ['request']


def request(method="GET", url="", **kwargs):
    if method == "GET":
        return 11111111
    return 22222222

```

```python
# test.py

import proxy.monkey

proxy.monkey.patch_all()

from app.libs.http_client import request

print(request(method="POST"))

```

output:
```text
22222222
```

是不是很酷，非常优雅的弥补了装饰器的不足。在测试的时候开启替换，然后就可以规避硬件或则网络等复杂因素。也让测试变得更加高效起来

参考

[捕蛇者说](https://pythonhunter.org/episodes/5)

[http://xiaorui.cc/archives/3248](http://xiaorui.cc/archives/3248)