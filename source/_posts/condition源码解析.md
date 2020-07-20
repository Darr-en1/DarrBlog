---
title: condition源码解析
copyright: true
permalink: 1
top: 0
date: 2020-07-20 10:42:47
tags:
    - python
    - condition
    - 源码
categories:
    - python
password:
---
Condition是一种多线程通信工具，表示多线程下参与数据竞争的线程的一种状态，主要负责多线程环境下对线程的挂起和唤醒工作<!--more-->

### 实例化

实例Condition时可以指定一个lock，如果没有指定，默认创建RLock的实例。同时Condition拥有与RLock一样的上锁方法acquire()和解锁方法release()。
并初始化双端队列waiters，用于存放 wait的thread

```python
class Condition:
    def __init__(self, lock=None):
        if lock is None:
            lock = RLock()
        self._lock = lock
        # Export the lock's acquire() and release() methods
        self.acquire = lock.acquire
        self.release = lock.release
        # 如果锁定义了_release_save()或_acquire_restore()，
        # 这些覆盖默认实现(只是调用Rlock的release()和acquire())。_is_owned同上()。        
        try:
            self._release_save = lock._release_save
        except AttributeError:
            pass
        try:
            self._acquire_restore = lock._acquire_restore
        except AttributeError:
            pass
        try:
            self._is_owned = lock._is_owned
        except AttributeError:
            pass
        self._waiters = _deque()
````
实现上下文语法
```python
    def __enter__(self):
        return self._lock.__enter__()

    def __exit__(self, *args):
        return self._lock.__exit__(*args)

```
### 初探notify和wait

条件锁的两个重要方法是notify()和wait()。notify()和wait()必须在条件锁上锁的状态下使用
notify()和wait()被调用，程序会先去调用self._is_owned()，判断当前线程号与条件锁中的self._ower是否一致，如果不一致，抛出异常RuntimeError。
```python
def _is_owned(self):
    # 如果当前线程拥有锁，_lock.acquire(0)返回False
    if self._lock.acquire(0):
        self._lock.release()
        return False
    else:
        return True
```

wait()先回释放第一层Rlock,其他线程就可以获取锁执行，然后会生成一个Lock,写入waiter并将自己锁住，等待其他线程释放 

```python
def wait(self, timeout=None):

    if not self._is_owned():
        raise RuntimeError("cannot wait on un-acquired lock")
    # 创建一个锁对象Lock，并获取它，然后把它放到waiting池，需要可以被其他线程释放
    waiter = _allocate_lock()
    waiter.acquire()
    self._waiters.append(waiter)
    # 释放底层锁，并保存锁对象的内部状态
    saved_state = self._release_save()
    gotit = False
    try:   
        if timeout is None:
            # 如果timeout是None，那么再次以阻塞的方式获取锁对象
            # 当前线程已经获取了一次该锁，再次acquire()当前线程会阻塞，直到其他线程释放该锁
            waiter.acquire()
            gotit = True
        else:
        # 如果timeout不是None，那么重复下面的流程：
                # time > 0 阻塞方式获取锁,timeout时间内block
                # time <0 以非阻塞方式获取锁
            if timeout > 0:
                gotit = waiter.acquire(True, timeout)
            else:
                gotit = waiter.acquire(False)
        return gotit
    finally:
        # 重新上第一层锁
        self._acquire_restore(saved_state)
        if not gotit:
            try:
                # 如果因为超时，而不是被唤醒，退出的wait()，那么将锁从waiting池中移除
                self._waiters.remove(waiter)
            except ValueError:
                pass
```

notify()是在一个双端队列中进行操作，这个队列在Condition中名为_waiters。默认情况下，notify只会释放一个锁（按先进先出原则）。
如果队列中没有锁，直接退出函数，不报任何异常。

```python
def notify(self, n=1):
    if not self._is_owned():
        raise RuntimeError("cannot notify on un-acquired lock")
    all_waiters = self._waiters
    waiters_to_notify = _deque(_islice(all_waiters, n))
    if not waiters_to_notify:
        return
    for waiter in waiters_to_notify:
        waiter.release()
        try:
            all_waiters.remove(waiter)
        except ValueError:
            pass
```

Condition中还有一个notify_all()方法，调用它会释放队列中全部的锁。

```python
def notify_all(self):
    self.notify(len(self._waiters))

notifyAll = notify_all
```

### 总结

condition 非常重要，在Queue,Semaphore,ThreadExecutorPool都有应用。

condition实现主要依靠两层锁：
 - condition初始化时创建一把锁(外部锁)，使用时需要先对外部锁上锁
 - 每次调用wait时，会先生成一个lock锁(内部锁)，将内部锁放到算双端队列waiters中，
    然后上锁，再将外部锁释放。并再次获取内部锁block,等待其他线程调用notify释放该内部锁，
    然后该线程会去试图获取内部锁。