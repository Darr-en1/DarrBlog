---
title: Redis数据结构底层_String
copyright: true
permalink: 1
top: 0
date: 2020-03-25 18:35:26
tags:
    - redis
    
categories:
    - redis
password:
---

String在Redis中是可变字节数组的形式存在,但它并不是C所提供的原始字符数组。Redis通过自建Simple dynamic string(SDS)实现。<!--more-->

#### SDS

redis在内部存储string都是用sds的数据结构实现。结构源码如下:
```chameleon
struct sdshdr{  
     //记录buf数组中已使用字节的数量,等于SDS保存字符串的长度  
     int len;  
     //记录buf数组中未使用字节的数量  
     int free;  
     //字节数组,用于保存字符串  
     char buf[];  
} 
```
redis的数据存储过程中为了提高性能，内部做了很多优化。string 内部还被拆分成三中编码:
 
 - int编码: 存储字符串长度小于20且能够转化为整数的字符串
 
 - embstr编码: 保存长度小于44字节的字符串(redis3.2版本之前是39字节，之后是44字节)
 
 - raw编码: 保存长度大于44字节的字符串(redis3.2版本之前是39字节，之后是44字节)

embstr编码的结构图：
![embstr编码的结构图](/images/redis-string/1.png)
raw编码的结构:
![raw编码的结构](/images/redis-string/2.png)

上面结构图可以看出embstr和raw都是由redisObject和sds组成的。
但是embstr的redisObject和sds是连续的,只需要使用malloc分配一次内存;
而raw需要为redisObject和sds分别分配内存，即需要分配两次内存。
因此数据量小的情况下，embstr显然更加效率。
但是当数据量变大,free不足,embstr因为是连续存储地址,需要对redisObject和sds重新分配,
而raw只需要sds分别分配内存。因此redis会将embstr转化成raw从而实现扩容。

Redis会自动对编码进行转换来适应和优化数据的存储,对 int(加减除外) embstr 的修改(append,setrange...)，
redis会先将其转换成raw，然后才进行修改。所以embstr实际上是只读性质的。


扩容策略：
 
    - string 容量 < 1 M ,扩大一倍
    - string 容量 > 1 M ,扩大 1 M
    - string 容量 MAX : 512M


参考

[https://redis.io/topics/internals-sds](https://redis.io/topics/internals-sds)

[http://blog.itpub.net/69918724/viewspace-2647282/](http://blog.itpub.net/69918724/viewspace-2647282/)

[https://database.51cto.com/art/201906/598234.htm](https://database.51cto.com/art/201906/598234.htm)

[https://database.51cto.com/art/201906/598234.htm](https://database.51cto.com/art/201906/598234.htm)

[https://www.jianshu.com/p/160fb0f73841](https://www.jianshu.com/p/160fb0f73841)
