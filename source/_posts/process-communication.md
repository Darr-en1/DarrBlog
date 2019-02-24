---
title: process communication
copyright: true
permalink: 1
top: 0
date: 2019-02-24 18:48:37
categories: python
tags: 
	- python
	- process
---
多进程之间是如何进行通信的呢？<!--more-->

#### 概述

##### 进程
进程是具有一定独立功能的程序关于某个数据集合上的一次运行活动,进程是系统进行资源分配和调度的一个独立单位.

##### 线程
线程是进程的一个实体,是CPU调度和分派的基本单位,它是比进程更小的能独立运行的基本单位.线程自己基本上不拥有系统资源,只拥有一点在运行中必不可少的资源(如程序计数器,一组寄存器和栈),但是它可与同属一个进程的其他的线程共享进程所拥有的全部资源.

##### 区别
进程和线程的主要差别在于它们是不同的操作系统资源管理方式。进程有独立的地址空间，一个进程崩溃后，在保护模式下不会对其它进程产生影响，而线程只是一个进程中的不同执行路径。线程有自己的堆栈和局部变量，但线程之间没有单独的地址空间，一个线程死掉就等于整个进程死掉，所以多进程的程序要比多线程的程序健壮，但在进程切换时，耗费资源较大，效率要差一些。

#### 多进程通信

##### 陈述 
每个进程都有独立的地址空间，设置全局变量并不能在多进程中共享

```python
import time
from multiprocessing import Process

def producer(a):
    a +=1
    time.sleep(2)
def consumer(a):
    time.sleep(2)
    print(a)

if __name__ =="__main__":
    a = 1
    my_producer = Process(target=producer,args=(a,))
    my_consumer = Process(target=consumer,args=(a,))
    my_producer.start()
    my_consumer.start()
    my_producer.join()
    my_consumer.join()
```
OutPut:  `1`

##### multiprocessing.Queue()

传统的Queue并不能满足多进程的使用

```python
import time

from multiprocessing import Process,Queue
def producer(queue):
    queue.put("a")
    time.sleep(2)
def consumer(queue):
    time.sleep(2)
    print(queue.get())

if __name__ =="__main__":
    queue = Queue(10)
    my_producer = Process(target=producer,args=(queue,))
    my_consumer = Process(target=consumer,args=(queue,))
    my_producer.start()
    my_consumer.start()
    my_producer.join()
    my_consumer.join()
```
OutPut:  `a`

##### multiprocessing.Pipe()
Pipe（）函数返回一对由管道连接的连接对象，只适用于两个进程，默认情况下是双工（双向）。

Pipe的性能高于queue

```python
import time

from multiprocessing import Pipe, Process


def producer(pipe):
    pipe.send("a")
    time.sleep(2)


def consumer(pipe):
    time.sleep(2)
    print(pipe.recv())


if __name__ == "__main__":
    recevie_pipe, send_pipe = Pipe()

    my_producer = Process(target=producer, args=(send_pipe,))
    my_consumer = Process(target=consumer, args=(recevie_pipe,))
    my_producer.start()
    my_consumer.start()
    my_producer.join()
    my_consumer.join()
```
OutPut:  `a`

##### multiprocessing.Manager()
通过Manager进行共享内存操作
```python
def add_data(p_dict,key,value):
    p_dict[key] = value

if __name__ =="__main__":
    process_dict = Manager().dict()
    first_process = Process(target=add_data,args=(process_dict,"Darr_en1",22))
    second_process = Process(target=add_data,args=(process_dict,"Darr_en2",44))
    first_process.start()
    second_process.start()
    first_process.join()
    second_process.join()

    print(process_dict)

```
OutPut:  `{'Darr_en1': 22, 'Darr_en2': 44}`