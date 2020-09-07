---
title: python collections 深入解析
copyright: true
permalink: 1
top: 0
date: 2020-09-05 18:20:54
categories: python
tags: 
	- python
	- collections
	- linked list
	- lru
password:
---


collections是Python内建的一个集合模块，提供了许多有用的集合类。<!--more-->


|  名称   |  说明   | 
|  ----   |  ----   | 
| [namedtuple](https://docs.python.org/zh-cn/3/library/collections.html#collections.namedtuple)    | 创建命名元组子类的工厂函数 |
| [deque](https://docs.python.org/zh-cn/3/library/collections.html#collections.deque)    | 类似列表(list)的容器，实现了在两端快速添加(append)和弹出(pop) |
| [ChainMap](https://docs.python.org/zh-cn/3/library/collections.html#collections.ChainMap)    | 类似字典(dict)的容器类，将多个映射集合到一个视图里面 |
| [Counter](https://docs.python.org/zh-cn/3/library/collections.html#collections.Counter)    | 	字典的子类，提供了可哈希对象的计数功能 |
| [OrderedDict](https://docs.python.org/zh-cn/3/library/collections.html#collections.OrderedDict)    | 字典的子类，保存了他们被添加的顺序 |
| [defaultdict](https://docs.python.org/zh-cn/3/library/collections.html#collections.defaultdict)    | 字典的子类，提供了一个工厂函数，为字典查询提供一个默认值 |
| [UserDict](https://docs.python.org/zh-cn/3/library/collections.html#collections.UserDict)    | 封装了字典对象，简化了字典子类化 |
| [UserList](https://docs.python.org/zh-cn/3/library/collections.html#collections.UserList)    | 封装了列表对象，简化了列表子类化 |
| [UserString](https://docs.python.org/zh-cn/3/library/collections.html#collections.UserString)    | 封装了列表对象，简化了字符串子类化 |

UserDict,UserList,UserString都是对于dict,list，str的封装，本质是一样的。
因为底层的内容可以作为属性来访问,一般用作继承从而实现自定义功能，不做过多概述。


### ChainMap

在Python中，当我们有两个字典需要合并的时候，可以使用字典的update方法，但这会改变其中一个字典。
如果我们不想改变原有的两个字典，那就需要单独再创建一个字典，但这会额外内存。
ChainMap可以在不修改原有字典前提下，又不另外创建一个新的字典，读写这个对象就像是读字典一样。
实际上ChainMap存储的是字典的引用。在其内部会储存一个Key到每个字典的映射，当你读取e[key]的时候，它先去查询这个key在哪个字典里面，然后再去对应的字典里面查询对应的值。

使用ChainMap需知:

- 如果两个字典里面有一个Key的名字相同，那么ChainMap会使用第一个拥有这个Key的字典里面的值
- 如果为ChainMap对象添加一个Key-Value对，新的Key-Value会被添加进第一个字典里面
- 如果修改原字典，修改的key在第一个字典或则在多个字典中只有一个，那么ChainMap对象也会相应更新。
如果这个Key在多个字典中都存在，且修改的不是第一个，则不更新。
- 如果从ChainMap对象里面删除一个Key，如果这个Key只在一个源字典中存在，
那么这个Key会被从源字典中删除。如果这个Key在多个字典中都存在，那么Key会被从第一个字典中删除。
当被从第一个字典中删除以后，第二个源字典的Key可以继续被ChainMap读取。


基本实现如下
```python
class ChainMap(MutableMapping):
    def __init__(self, *maps):
        self.maps = list(maps) or [{}]

    def __getitem__(self, key):
        for mapping in self.maps:
            try:
                return mapping[key]
            except KeyError:
                pass
        return self.__missing__(key)

    def __setitem__(self, key, value):
        self.maps[0][key] = value
    
    def __delitem__(self, key):
        try:
            del self.maps[0][key]
        except KeyError:
            raise KeyError('Key not found in the first mapping: {!r}'.format(key))
```

### Counter

Counter是dict的子类，用于计数可哈希对象。它是一个集合，元素像字典键(key)一样存储，它们的计数存储为值。计数可以是任何整数值，包括0和负数。

主要实现逻辑在update方法中:

```python
class Counter(dict):
    def __init__(*args, **kwds):
        if not args:
            raise TypeError("descriptor '__init__' of 'Counter' object "
                            "needs an argument")
        self, *args = args
        if len(args) > 1:
            raise TypeError('expected at most 1 arguments, got %d' % len(args))
        super(Counter, self).__init__()
        self.update(*args, **kwds)
        
    def update(*args, **kwds):
        if not args:
            raise TypeError("descriptor 'update' of 'Counter' object "
                            "needs an argument")
        self, *args = args
        if len(args) > 1:
            raise TypeError('expected at most 1 arguments, got %d' % len(args))
        iterable = args[0] if args else None
        if iterable is not None:
            if isinstance(iterable, Mapping):
                if self:
                    self_get = self.get
                    for elem, count in iterable.items():
                        self[elem] = count + self_get(elem, 0)
                else:
                    super(Counter, self).update(iterable) # fast path when counter is empty
            else:
                _count_elements(self, iterable)
        if kwds:
            self.update(kwds)
            
def _count_elements(mapping, iterable):
    'Tally elements from the iterable.'
    mapping_get = mapping.get
    for elem in iterable:
        mapping[elem] = mapping_get(elem, 0) + 1
```

通过源码分析：Counter可以传入iterable或则Mapping。
当传入 iterable时，会调用_count_elements，遍历累加元素个数
当传入 Mapping时，会遍历累加`self[elem] = count + self_get(elem, 0)`，通过源码可以发现，传入的Mapping的value值必须为整数。

### deque
deque作为双端队列,其底层实现基于双向链表,其插入和删除效率远高于list,因此可以用来实现栈（stack）也可以用来实现队列（queue）。

双向链表实现:
```python
class Node(object):
    def __init__(self, value=None, prev=None, next=None):
        self.value, self.prev, self.next = value, prev, next

    def __str__(self):
        return f"{self.__class__}: {self.value}"


class CircularDoubleLinkedList(object):
    def __init__(self, maxsize=20):
        self.maxsize = maxsize
        node = Node()
        node.next, node.prev = node, node
        self.root = node
        self.length = 0

    def __len__(self):
        return self.length

    def __str__(self):
        return f"{self.__class__}: {self.tolist()}"

    @property
    def headnode(self):
        return self.root.next

    @property
    def tailnode(self):
        return self.root.prev

    def append(self, value):
        self.is_full()
        node = Node(value, prev=self.tailnode, next=self.root)
        self.tailnode.next = node
        self.root.prev = node
        self.length += 1

    def pop(self):

        tailnode = self.tailnode

        if tailnode == self.root:
            raise Exception("Empty")

        tailnode.prev.next = self.root
        self.root.prev = tailnode.prev
        self.length -= 1

        return tailnode.value

    def pop_left(self):

        headnode = self.headnode

        if headnode == self.root:
            raise Exception("Empty")

        headnode.next.prev = self.root
        self.root.next = headnode.next
        self.length -= 1

        return headnode.value

    def append_left(self, value):

        if self.root.next is self.root:
            return self.append(value)

        self.is_full()

        headnode = self.headnode

        node = Node(value, prev=self.root, next=headnode)
        self.root.next = node
        headnode.prev = node

        self.length += 1

    def remove(self, node):
        if node is self.root:
            return

        node.prev.next = node.next
        node.next.prev = node.prev
        self.length -= 1
        del node
        return 1

    def iter_node_reverse(self):
        if self.root.prev is self.root:
            return

        node = self.root.prev

        while node is not None and node != self.headnode:
            yield node
            node = node.prev
        yield node

    def __iter__(self):
        for node in self.iter_node():
            yield node.value

    def iter_node(self):

        if self.root.next is self.root:
            return

        node = self.root.next

        while node is not None and node != self.tailnode:
            yield node
            node = node.next
        yield node

    def tolist(self):
        return [i for i in self]

    def is_full(self):
        if len(self) > self.maxsize:
            raise Exception("Full")
```

### namedtuple

collection.namedtuple能够方便地在Python中手动定义一个内存占用较少的不可变类。


类实现:
```python
class User:
    def __init__(self, name, age):
        self.name = name
        self.age = age
```

namedtuple实现:
```python
User = namedtuple("User", ["name", "age", "height"])
```

namedtuple转换为dict:
```python
user_info_dict = user._asdict()
```

namedtuple拆包:
```python
name, age, *others = user
```

### defaultdict

defaultdict属于内建函数dict的一个子类，调用工厂函数提供缺失的值。


#### 对比

假如我们要对一个字符串字符计数

```python
import random
import string
# 对_str字符计数
_str = ''.join(random.choices(string.ascii_lowercase, k=20))
```

**usual**:

```python
counter = {}
for s in _str:
    try:     
        counter[s] += 1
    except KeyError:
        counter[s] = 1
```

**setdefault**:

```python
counter = {}
for s in _str:
    counter.setdefault(s, 0)
    counter[s] += 1
```

**defaultdict**:

```python
counter = defaultdict(int)
for s in _str:
    counter[s] += 1
```

#### 原理

defaultdict是怎么实现这一功能的呢，我们可以看一下UserDict源码的```__getitem__```:

```python
 def __getitem__(self, key):
        if key in self.data:
            return self.data[key]
        if hasattr(self.__class__, "__missing__"):
            return self.__class__.__missing__(self, key)
        raise KeyError(key)
```
可以看到当dict中不存在某一key时，会试图执行```__mising__```方法，没有则向上抛出异常。

因此我们可以自定义defaultdict:
```python
from collections import UserDict


class CustomDefaultDict(UserDict):
    def __init__(self, default_factory=None, *arg, **kwargs):
        super().__init__(*arg, **kwargs)
        self.default_factory = default_factory

    def __missing__(self, key):
        self[key] = value = self.default_factory()
        return value
```

### OrderedDict
dict基于hash实现，具备优秀的插入效率和查询效率，但却无法做到有序插入。
在一些需要插入顺序的场景下显得乏力。怎么做到，既保证其时序性的同时又具备很好的检索效率呢，
我们可以设计一个链表，用于存储key插入的先后顺序。这就是OrderedDict的是实现原理。

#### 源码解析
OrderedDict在python中通过双向链表和dict实现。在python 3.6 版本中，dict默认具备插入顺序。

通过阅读源码我们可以看到，OrderedDict定义了两个实例变量```self.__root```,```self.__map```。

```self.__root```做为链表的头节点用于实现循环双向链表

```self.__map```将字典的key与双向链表中节点进行关联

借助这两个变量OrderedDict的实现就很简单了

```python
class OrderedDict(dict):
    def __init__(*args, **kwds):
        if not args:
            raise TypeError("descriptor '__init__' of 'OrderedDict' object "
                            "needs an argument")
        self, *args = args
        if len(args) > 1:
            raise TypeError('expected at most 1 arguments, got %d' % len(args))
        try:
            self.__root
        except AttributeError:
            self.__hardroot = _Link()
            self.__root = root = _proxy(self.__hardroot)
            root.prev = root.next = root
            self.__map = {}
        self.__update(*args, **kwds)
        
    def __setitem__(self, key, value,
                    dict_setitem=dict.__setitem__, proxy=_proxy, Link=_Link):
        if key not in self:
            self.__map[key] = link = Link()
            root = self.__root
            last = root.prev
            link.prev, link.next, link.key = last, root, key
            last.next = link
            root.prev = proxy(link)
        dict_setitem(self, key, value)

    def __delitem__(self, key, dict_delitem=dict.__delitem__):
        dict_delitem(self, key)
        link = self.__map.pop(key)
        link_prev = link.prev
        link_next = link.next
        link_prev.next = link_next
        link_next.prev = link_prev
        link.prev = None
        link.next = None

    def __iter__(self):
        root = self.__root
        curr = root.next
        while curr is not root:
            yield curr.key
            curr = curr.next
```

#### LRU算法实现

在有限的空间中存储对象时，当空间满时，会按照一定算法从有限空间删除部分对象。
常用的算法有LRU，FIFO，LFU等。
LRU：least recently used，最近最少使用算法。在计算机的Cache硬件，以及主存到虚拟内存的页面置换，还有Redis缓存系统中都用到了该算法。

在Python中，可以使用collections.OrderedDict很方便的实现LRU算法


```python
from collections import OrderedDict
 
 
class LRUCache(OrderedDict):

    def __init__(self,capacity):
        self.capacity = capacity
        self.cache = OrderedDict()
     

    def get(self,key):
        if self.cache.has_key(key):
            value = self.cache.pop(key)
            self.cache[key] = value
        else:
            value = None
         
        return value
     

    def set(self,key,value):
        if self.cache.has_key(key):
            value = self.cache.pop(key)
            self.cache[key] = value
        else:
            if len(self.cache) == self.capacity:
                self.cache.popitem(last = False)
                self.cache[key] = value
            else:
                self.cache[key] = value

```

参考:
[https://juejin.im/post/6844903875711860750](https://juejin.im/post/6844903875711860750)







