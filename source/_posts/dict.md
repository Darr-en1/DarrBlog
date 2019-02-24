---
title: __dict__
date: 2019-02-24 19:07:35
tags: python
categories: python
password: 
---
Python下一切皆对象，每个对象都有多个属性(attribute)，Python对属性有一套统一的管理方案。__dict__是用来存储对象属性的一个字典，其键为属性名，值为属性的值。<!--more-->

#### __dict__访问对象属性 
__dict__属性是一个字典(dict)，它包含了该对象所有的属性。
```python
class Parent(object):
    a = 0
    b = 1

    def __init__(self):
        self.a = 2
        self.b = 3

    def p_test(self):
        pass



p = Parent()
print(Parent.__dict__)
print(p.__dict__)
```
OutPut:
```python
{'__module__': '__main__', 'a': 0, 'b': 1, '__init__': <function Parent.__init__ at 0x0000021251F037B8>, 'p_test': <function Parent.p_test at 0x000002125218DAE8>, '__dict__': <attribute '__dict__' of 'Parent' objects>, '__weakref__': <attribute '__weakref__' of 'Parent' objects>, '__doc__': None}
{'a': 2, 'b': 3}
```

python中的内置的数据类型（如int, list, dict）不存在__dict__属性

#### 继承状态下__dict__属性

子类有自己的__dict__,父类也有自己的__dict__,子类的类变量和函数放在子类的dict中，父类的放在父类dict中。

父子类对象有自己的__dict__属性， 存储self.xx 信息，子类不重写父类实例化变量，父子类对象__dict__一致，子类重写父类实例化变量，子类对象__dict__会发生改变

```python
class Parent(object):
    a = 0
    b = 1

    def __init__(self):
        self.a = 2
        self.b = 3

    def p_test(self):
        pass


class Child(Parent):
    # a = 4
    # b = 5

    def __init__(self):
        super(Child, self).__init__()
        # self.a = 6
        # self.b = 7

    def c_test(self):
        pass

    def p_test(self):
        pass

p = Parent()
c = Child()
print(Parent.__dict__)
print(Child.__dict__)
print(p.__dict__)
print(c.__dict__)
```

```python
{'__module__': '__main__', 'a': 0, 'b': 1, '__init__': <function Parent.__init__ at 0x000001FC798437B8>, 'p_test': <function Parent.p_test at 0x000001FC79ACDAE8>, '__dict__': <attribute '__dict__' of 'Parent' objects>, '__weakref__': <attribute '__weakref__' of 'Parent' objects>, '__doc__': None}
{'__module__': '__main__', '__init__': <function Child.__init__ at 0x000001FC79ACDB70>, 'c_test': <function Child.c_test at 0x000001FC79ACDBF8>, 'p_test': <function Child.p_test at 0x000001FC79ACDC80>, '__doc__': None}
{'a': 2, 'b': 3}
{'a': 2, 'b': 3}
```

#### __dict__操作私有变量

java中可以通过反射机制操作到对象的私有属性,python中可以通过__dict__操作私有变量

__xx：双前置下划线，私有化属性或方法，无法在外部直接访问（名字重整所以访问不到）
```python
class test(object):
    def __init__(self):
        self.num = 10
        self.__num = 30

    def __str__(self):
        return f"num = {self.num},__num = {self.__num}"

t = test()
print(t.num)    # 10
# print(t.__num)  # AttributeError: 'test' object has no attribute '__num'
print(t.__dict__['_test__num'])
t.__dict__['num'] = 50
t.__dict__['_test__num'] = 100
print(t)
```
OutPut:
```python
10
30
num = 50,__num = 100
```


