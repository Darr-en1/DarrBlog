---
title: redis引用之计数器
copyright: true
permalink: 1
top: 0
date: 2020-04-03 18:16:36
tags:
    - redis
categories:
    - redis
password:
---

当我们的网站上线之后,我们很多时候需要对网站浏览记录进行分析，从而做出对策。
如:我们需要对热点页面进行缓存，因此我们就需要知道页面点击数等等。
由于redis的处理命令做到线程安全，并且支持大量的读写操作，因此将计数器储存到redis里面是一个非常好的方案。<!--more-->

## 设计思路

我们需要记录不同时间精度下的用户点击数，首先相同精度只需存储单次，其实我们需要可以遍列表获取精度,
显然需要具备去重并且排序的特性。这里我们选择zset用于存储精度数据。将分值设为0，使其根据成员名进行排序。
```python
# key：known
# type:zset

1:hits    0
5:hits    0
60:hits    0
60*60:hits    0
```


而对于某一个精度下的点击数量的记录，我们采用hash 来进行存储，key 为时间戳可以保证唯一性，对value 进行incr ,
由于redis 的单线程特性，已经保证对操作的原子性。

```python
# key: count:1:hits  
# type:hash

time1    70
time2    40
time3    50
time4    50
```

## 对计数器进行更新

通过pipeline既可以减少通信次数，还可以保证这些命令执行过程中不会插入其他命令。

```python
import time

PRECISION = (1, 5, 60, 5 * 60, 60 * 60, 5 * 60 * 60, 24 * 60 * 60,)


def update_counter(conn, name, count=1, now=None):
    """
    :param conn:    redis连接对象
    :param name:    统计名(点击量，销量...)
    :param count:   访问数
    :param now:   当前时间
    :return:
    """

    now = now or time.time()
    pipe = conn.pipeline()
    for prec in PRECISION:
        # 取得当前精度的开始时间
        pnow = now // prec * prec
        hash = f'{prec}:{name}'
        pipe.zadd('known:', {hash: 0})
        pipe.hincrby('count:' + hash, pnow, count)
    pipe.execute()
```

### 获取指定精度的数据

通过获取precision精度hash 表下的所有数据，并对时间戳进行排序并格式化，展示给用户。

```python
def get_counter(conn, name, precision, data_format="%Y-%m-%d %H:%M:%S"):
    hash = f'{precision}:{name}'
    all_counter = conn.hgetall('count:' + hash)
    return sorted(
        map(lambda obj: (time.strftime(data_format, time.localtime(int(obj[0]))), int(obj[1])), all_counter.items()),
        key=lambda obj: (obj[0], obj[1],))
```

### 清理旧计数器

在redis中，针对zset并没有对于其内部key对应的expire操作，因此需要提供一个删除解决方案。

需要注意以下几点：
- 在删除过程中，随时可能有新的计数器更新或添加进来，因此在删除计数器的时候需要使用redis事务。
- 针对一些更新频率过长的计数器，可以降低清理频率。



```python
QUIT = False

def clean_counters(conn):
    pipe = conn.pipeline(True)
    # 程序清理操作执行的次数,每执行一次加一
    passes = 0
    # 持续地对计数器进行清理，直到退出为止。
    while not QUIT:
        # 记录清理操作开始执行的时间，用于计算清理操作执行的时长。
        start = time.time()
        # 作为遍历 known(zset)表 value 的 index
        index = 0
        while index < conn.zcard('known:'):
            # 取特定精度的计数器表 key (hash)
            hash = conn.zrange('known:', index, index)
            index += 1
            if not hash:
                break
            hash = hash[0]
            # 取得计数器的精度。
            prec = int(hash.partition(':')[0])
            # 通过精度计算清理的频率(精度为 1 min 以内的设置每次轮询都会清理，大于1min 设置 int(prec // 60)次进行清理)
            bprec = int(prec // 60) or 1
            # 判断当前精度处于当前轮询次数下是否需要被清理
            if passes % bprec:
                continue

            hkey = 'count:' + hash
            # 计算该  间
            cutoff = time.time() - SAMPLE_COUNT * prec
            # 获取该精度下所有样本并排序
            samples = sorted(map(int, conn.hkeys(hkey)))
            # 通过二分查找已经排序的列表中过期的key
            remove = bisect.bisect_right(samples, cutoff)

            if remove:
                # 移除过期计数样本
                conn.hdel(hkey, *samples[:remove])
                # 判断计数样本是否全部过期，过期删除在zset中的内容
                if remove == len(samples):
                    try:
                        # 在尝试修改计数器散列之前，对其进行监视。确保删除时有添加可以拒绝删除操作
                        pipe.watch(hkey)
                        # 验证计数器散列是否为空，如果是的话，那么从记录已知计数器的有序集合里面移除它
                        if not pipe.hlen(hkey):
                            pipe.multi()
                            pipe.zrem('known:', hash)
                            pipe.execute()
                            # 删除一个计数器，zset的zcard减一,因此index不变即可获取下一个精度
                            index -= 1
                        else:
                            # 计数器散列并不为空，继续让它留在记录已有计数器的有序集合里面
                            pipe.unwatch()
                    # 删除过程中有其他程序向这个计算器散列添加了新的数据，继续让它留在记录已知计数器的有序集合里面
                    except WatchError:
                        pass

        # 清理次数加一
        passes += 1
        duration = min(int(time.time() - start) + 1, 60)
        # 如果这次循环未耗尽60秒钟，那么在余下的时间内进行休眠；
        # 如果60秒钟已经耗尽，那么休眠一秒钟以便稍作休息。
        time.sleep(max(60 - duration, 1))
```

以上就是通过redis 实现一个简单的计数器功能。
