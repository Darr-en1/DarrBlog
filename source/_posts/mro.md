---
title: python中的多继承
copyright: true
permalink: 1
top: 0
date: 2020-04-26 18:04:08
tags: 
    - mro
    - python
categories: python
password:
---
相信学过java的同学们都知道java并不支持多继承，但python中却是支持多继承的，而且像python社区中非常有名的框架中就疯狂引入多继承来扩展类的功能。python多继承基于mro算法来实现的。
<!--more-->

### MRO 发展史

**二义性问题**：当b1,b2都继承A时，假设A类有个f()方法都被b1,b2重写。此时c在同时继承b1,b2当调用f()方法时，就无法确定调用那个方法。

python 为了处理二义性问题，Python采用MRO机制。在python发展过程也逐步对MRO算法进行优化。

#### before python2.2

**Python2.2以前的版本：经典类**

**MRO算法依据:DFS**

DFS(深度优先算法)采用递归的形式，用到了栈结构，先进后出，存在一定缺陷。

![DFS继承](/images/mro/1.png)

在正常继承模式下，符合预期，但是在棱形继承模式下C类即便继承D类的方法，但由于D类在DFS下先于C类，因此C类的方法永远不会被执行

#### python2.2

**python2.2开始引入新式类，经典类依旧采用DFS,而新式类采用BFS**


[python新式类和经典类的区别](https://www.zhihu.com/question/22475395)

![BFS继承](/images/mro/2.png)

BFS(广度优先算法)选取状态用队列的形式，先进先出，依旧存在缺陷。虽然棱形继承模式下的缺陷被解决了，但正常继承模式下显得不太常规。



#### python2.3~2.7
从python2.3开始python对新式类的MRO算法进行调整,采用C3算法。一举解决了DFS BFS 存在的局限。

![C3继承](/images/mro/3.png)

#### after python3
python3开始,python放弃了经典类，所有的类都是新式类采用C3算法。

#### 总结

|  python版本   | MRO算法依据  |
|  ----  | ----  |
| before python2.2  | DFS(经典类) |
| python2.2  | DFS(经典类),BFS(新式类) |
| python2.3~2.7  | DFS(经典类),C3(新式类) |
| python3  | C3(新式类) |

### C3 线性算法原理
我们用 C1C2⋯CN表示包含 N 个类的列表，并令
head(C1C2⋯CN)=C1 
tail(C1C2⋯CN)=C2C3⋯CN
假设类 C继承自父类 B1,⋯,BNB1,⋯,BN，那么根据 C3 线性化，类 C的方法解析列表通过如下公式确定：
L[C(B1⋯BN)]=C+merge(L[B1],⋯,L[BN],B1⋯BN)


该操作可以分为以下几个步骤:
1.选取 merge 中的第一个列表记为当前列表 K。
2.令 h=head(K) ，如果 h没有出现在其他任何列表的 tail 当中，那么将其加入到类 C 的线性化列表中，并将其从 merge 中所有列表中移除，之后重复步骤 2。
3.否则，设置 KK 为 merge 中的下一个列表，并重复 2 中的操作。
4.如果 merge 中所有的类都被移除，则输出类创建成功；如果不能找到下一个 h，则输出拒绝创建类 CC 并抛出异常。


![继承关系图](/images/mro/4.png)

算法演算如下：

```python
L[E] = L[E(O)] 
     = E + merge(L[O],O) 
     = [E,O]
L[D] = [D,O]
L[F] = [F,O]
L[B] = B + merge(L[E],L[D],E,D) 
     = B + merge([E,O],[D,O],E,D) 
     = [B,E,D,O]
L[C] = C + merge(L[D],L[F],D,F) 
     = [C,D,F,O]
L[A] = A + merge([B,E,D,O],[C,D,F,O],B,C) 
     = [A,B,E,C,D,F,O]

```

查看MRO 继承关系： `__MRO__`

以下是本人之前写的ppt,提供参考:

[MRO_PPT](https://github.com/Darr-en1/ppt/blob/master/Python%E5%A4%9A%E9%87%8D%E7%BB%A7%E6%89%BF_MRO.pptx)