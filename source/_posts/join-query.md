---
title: 组合两个表
copyright: true
permalink: 1
top: 0
date: 2019-04-10 15:19:26
tags:
    - mysql
    - leetcode
    - join
categories: mysql
password:
---


通过连接查询取得数据库字段<!--more-->

**Person表**

列名  | 类型
---|---
PersonId | int （主）
FirstName | varchar 
LastName | varchar 

**Address表**

列名 | 类型 
---|---
AddressId | int（主） 
PersonId | int （外）
City | varchar 
State | varchar 

**题目**

编写一个SQL查询，满足条件：无论person是否有地址信息，都需要基于上述两表提供
person的以下信息：

FirstName, LastName, City, State

**本题目考察的知识点有两点，关联查询，on where区别**

数据库在通过连接两张或多张表来返回记录时，都会生成一张中间的临时表，然后再将这张临时表返回给用户。

本题中**无论person是否有地址信息，都会返回person信息**，所以选用左外连接（left join）。

on 和where区别：

1.on条件是在生成临时表时使用的条件，它不管on中的条件是否为真，都会返回左边表中的记录。
 
2.where条件是在临时表生成好后，再对临时表进行过滤的条件。这时已经没有left join的含义（必须返回左边表的记录）了，条件不为真的就全部过滤掉。

3.因为inner join取并集，因此on where 没区别


**答案**
```mysql
SELECT p.FirstName, p.LastName, a.City, a.State 
FROM Person p LEFT JOIN Address a 
ON  p.PersonId = a.personId;
```



[on与where区别](https://www.cnblogs.com/toSeeMyDream/p/6843984.html)