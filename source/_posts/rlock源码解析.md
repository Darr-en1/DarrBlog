---
title: rlock源码解析
copyright: true
permalink: 1
top: 0
date: 2020-07-07 14:48:12
tags:
    - python
    - lock
    - 源码
categories:
    - python
password:
---

可重入锁RLock允许在**同一线程**中被多次**acquire**，我们将一一揭晓它的实现过程。<!--more-->

#### Lock

```python
import _thread


_allocate_lock = _thread.allocate_lock

Lock = _allocate_lock
```

Lock只是简单的对底层库做了个代理。**锁并不属于锁定它的线程;另一个线程可以解锁它**。由C代码实现。

```python
import threading
import time


def worker1(lock):
    lock.acquire()
    print("worker1 running")
    time.sleep(5)
    lock.release()
    pass


def worker2(lock):
    lock.release()
    print("worker2 running")
    time.sleep(2)
    lock.acquire()
    pass


lock = threading.Lock()
t = threading.Thread(target=worker1, args=(lock,))
t1 = threading.Thread(target=worker2, args=(lock,))
t.start()
t1.start()
```
worker2可以释放worker1获得的锁，获得的锁也能被worker1释放

#### RLock

两个特性:
- 可重入:持有锁的线程可以多次acquire,release
- 线程独占:当前线程获取的锁其他线程不能释放

```python
def RLock(*args, **kwargs):
    """
    该工厂函数返回一个新的可重入锁。
    一个可重入锁必须由创建它的线程释放。一旦一个线程获得了一个可重入锁，该线程可用无阻塞的再次获取。每次获取锁后必须进行释放。
    """
    if _CRLock is None:
        return _PyRLock(*args, **kwargs)
    return _CRLock(*args, **kwargs)
```
在3.6.5版本中。默认采用的是C 语言实现的，即_CRLock。还有一个Python版本实现的，即_PyRLock（也就是类_RLock）

```python
class _RLock:

    def __init__(self):
        self._block = _allocate_lock()
        self._owner = None
        self._count = 0
```

通过查看Rlock 实例化可以得知，Rlock只是在Lock基础上另外维护了_owner,_count从而达到-可重入的效果；

_block :内部互斥排他锁同Lock
_owner :记录RLock锁的持有者的线程id,判断锁的持有者和当前的线程是否一致
_count : 内部的计数器，用来记录锁被持有的次数，对于RLock的属主线程，每获取一次锁就+1，反之每释放一次锁就-1，当计数器为0时，就释放内部的锁，这样其他线程可以继续获取该内部锁，继而获取了这个Rlock


```python
def acquire(self, blocking=True, timeout=-1):
        me = get_ident() 
        if self._owner == me:   # 如果当前线程的pid是RLock对象所在的线程，那么对计数器进行加一操作
            self._count += 1
            return 1
        # 不满足上述条件：
        # 1. 当前线程非RLock对象所在线程
        # 2. RLock对象还未持有锁，即 self.owner = None
        
        # 对Lock上锁：
        # 当 blocking=True 时，当前线程被阻塞，直到持有锁的线程将锁释放后，rc = True
        # 当 blocking=False 时，可以非阻塞的获取。如果获取锁成功，rc = True；获取失败，rc = False
        rc = self._block.acquire(blocking, timeout)
        if rc:
            self._owner = me   
            self._count = 1   
        return rc
```

acquire方法的流程图如下:
![acquire方法的流程图](/images/rlock源码解析/1.png)


```python
def release(self):
        if self._owner != get_ident():  # 如果持有锁的线程非当前线程，则抛异常
            raise RuntimeError("cannot release un-acquired lock")
        self._count = count = self._count - 1  
        if not count:   # 如果计数器减到0，那么释放RLock内部的锁，此时其他线程就可以获取到锁
            self._owner = None   
            self._block.release()
```

release方法的流程图如下:
![release方法的流程图](/images/rlock源码解析/2.png)



参考：

[https://blog.csdn.net/u010301542/article/details/80144643](https://blog.csdn.net/u010301542/article/details/80144643)

[https://reishin.me/python-source-code-parse-with-rlock/](https://reishin.me/python-source-code-parse-with-rlock/)

[https://www.cnblogs.com/buwu/p/12740646.html](https://www.cnblogs.com/buwu/p/12740646.html)