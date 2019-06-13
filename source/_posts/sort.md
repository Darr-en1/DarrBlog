---
title: sort
copyright: true
permalink: 1
top: 0
date: 2019-05-08 14:22:45
tags: 
    - python
    - sort
categories: python
password:
---

排序在编程中非常常见，python内置sort()和sorted()两中排序方式。<!--more-->
<!--more-->
### list.sort() sorted() 区别

```python

# python3 取消cmp
sorted(iterable, key=None, reverse=False)  
list.sort(self, key=None, reverse=False)
```

sort()与sorted()的不同在于，sort是在原位重新排列list， sort() 函数不需要复制原有列表，消耗的内存较少，效率也较高。

sorted()是产生一个新的列表，并可以接受任何形式的可迭代对象（包括不可变序列和生成器），并返回list，因此可以满足不同需求。

### 介绍

参数用法大体一致
#### sorted()

iterable：是可迭代类型;
key：用列表元素的某个属性和函数进行作为关键字，有默认值，迭代集合中的一项;

reverse：排序规则. reverse = True 或者 reverse = False，有默认值；

return：list

#### list.sort()

sort() 是list的内置函数通过list.sort()调用

key：用列表元素的某个属性和函数进行作为关键字，有默认值，迭代集合中的一项;

reverse：排序规则. reverse = True 或者 reverse = False，有默认值。 

return：None


#### 对str去重排序
```python
# sort()
In [2]: str="cdsciolsdruikcadssv"
   ...: s = list(set(str))
   ...: s.sort(reverse=False)
   ...: print("".join(s))
acdiklorsuv

# sorted()
In [3]: print("".join(sorted(set(str),reverse=False)))
acdiklorsuv
```

### sort 之魅

给定一个只包含大小写字母，数字的字符串，对其进行排序，保证：

所有的小写字母在大写字母前面
所有的字母在数字前面
所有的奇数在偶数前面

```python
In [5]: s = "Sorting1234"
   ...: "".join(sorted(s, key=lambda x: (x.isdigit(), x.isdigit() and int(x) % 2 == 0, x.isupper(), x.islower(), x)))
Out[5]: 'ginortS1324'
```

分析：
key 用来决定在排序算法中 cmp 比较的内容，key 可以是任何可被比较的内容，lambda 函数将输入的字符转换为一个元组，然后 sorted 函数将根据元组（而不是字符）来进行比较，进而判断每个字符的前后顺序。

元组大小比较：将第一元组的第一项与第二元组的第一项进行比较;如果它们不相等，这是比较的结果，否则第二项被考虑，然后第三项，等等。

### sort函数内部实现原理

sort 内部通过Timsort实现，详情查看[python sort函数内部实现原理](https://www.cnblogs.com/clement-jiao/p/9243066.html)



