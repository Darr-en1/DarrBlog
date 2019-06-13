---
title: middleware
copyright: true
permalink: 1
top: 0
date: 2019-06-13 11:08:41
tags:
    - python
    - Django
categories: python
password:
---

middleware与Django的请求/响应生命周期挂钩，middleware中定义的钩子函数可以用于全局改变Django的输入或输出。

本文将从以下几点介绍Django的middleware：

- 什么时候使用 middleware
- middleware是怎么运作的
- 自定义 StackOverflowMiddleware
<!--more-->

### 什么时候使用 middleware

当你需要修改请求，例如被传送到view中的HttpRequest对象。 或者修改view返回的HttpResponse对象，亦或者对运行中的异常作通用处理，使用中间件是一个很好的解决方案。

本人之前学习javaWeb时接触到filter可以用来识别用户是否登录，Django中的middleware和这个功能非常相似，可以在请求调用视图前做请求处理，从而实现校验逻辑。

#### 全局拦截

统计一分钟访问页面数，对于访问次数过多的 IP 将其到黑名单 BLOCKED_IPS，并拒绝访问

```python
class BlockedIpMiddleware(object):
    def process_request(self, request):
        if request.META['REMOTE_ADDR'] in getattr(settings, "BLOCKED_IPS", []):
            return http.HttpResponseForbidden('<h1>Forbidden</h1>')
```
#### 错误显示

网站放到服务器上正式运行后，DEBUG改为了False，这样更安全，但是有时候发生错误我们不能看到错误详情，可以设置管理员看到的是错误详情，而正常用户则显示友好的报错信息。

```python

import sys
from django.views.debug import technical_500_response
from django.conf import settings
class UserBasedExceptionMiddleware(object):
    def process_exception(self, request, exception):
        if request.user.is_superuser or request.META.get('REMOTE_ADDR') in settings.INTERNAL_IPS:
            return technical_500_response(request, *sys.exc_info())
```

### middleware是怎么运作的

我们从浏览器发出一个请求 Request，得到一个响应后的内容 HttpResponse ，这个请求传递到 Django的过程如下：

![image](https://docs.djangoproject.com/en/1.8/_images/middleware.svg)

Django 提供了一下五种钩子函数：
```python
process_request(self,request)
process_view(self, request, view_func, view_args, view_kwargs)
process_template_response(self,request,response)
process_exception(self, request, exception)
process_response(self, request, response)
```
想要使用middleware，需要在setting中的MIDDLEWARE中注册从而进行激活。middleware可以存在依赖关系，所以在注册时，需要确定依赖关系，从而排列合适的顺序。

上面的方法，前两个方法是请求进来时要穿越的，而后三个方法是请求出去时要穿越的，在MIDDLEWARE注册的middleware，请求是顺序执行的，响应是逆序执行的。

#### 总结

**中间件的定义**：wsgi之后 urls.py之前 在全局操作Django请求和响应的模块

#### 中间件的使用

##### process_request(self, request)

执行顺序：按照注册的顺序（在settings.py里面设置中 从上到下的顺序）

何时执行：请求从wsgi拿到之后
返回值：
- 返回None，继续执行后续的中间件的process_request方法
- 返回response , 不执行后续的中间件的process_request方法

##### process_response(self, request, response)

执行顺序：按照注册顺序的倒序（在settings.py里面设置中 从下到上的顺序）

何时执行：请求有响应的时候

返回值：必须返回一个response对象

##### process_view(self, request, view_func, view_args, view_kwargs)

执行顺序：按照注册的顺序（在settings.py里面设置中 从上到下的顺序）

何时执行：在urls.py中找到对应关系之后 在执行真正的视图函数之前

返回值：
- 返回None，继续执行后续的中间件的process_view方法
- 返回response,


##### process_exception(self, request, exception)

执行顺序：按照注册顺序的倒序（在settings.py里面设置中 从下到上的顺序）

何时执行：视图函数中抛出异常的时候才执行

返回值：
- 返回None,继续执行后续中间件的process_exception
- 返回response，

##### process_template_response(self, request, response)

执行顺序：按照注册顺序的倒序（在settings.py里面设置中 从下到上的顺序）

何时执行：视图函数执行完，在执行视图函数返回的响应对象的render方法之前 

返回值：
- 返回None,继续执行后续中间件的process_exception
- 返回response，


##### Django调用 注册的中间件里面五个方法的顺序
```md
1. process_request
urls.py
2. process_view
view
3. 有异常就执行 process_exception
4. 如果视图函数返回的响应对象有render方法,就执行process_template_response
5. process_response
```

### 自定义 StackOverflowMiddleware

本小节将会创建一个异常处理的Middleware，并接入StackOverflow APi 在Debug 模式下 终端返回最高票数的解决方案。

需要使用[requests](https://2.python-requests.org//en/master/) 和[StackOverflow API](https://api.stackexchange.com/docs)和python3.6运行环境

```python
# middleware.py

import requests
from django.utils.deprecation import MiddlewareMixin

__author__ = 'Darr_en1'


class StackOverflowMiddleware(MiddlewareMixin):

    def process_exception(self, request, exception):
        intitle = f'{exception.__class__.__name__}: {exception.args[0]}'

        url = 'https://api.stackexchange.com/2.2/search'
        headers = {'User-Agent': 'github.com/vitorfs/seot'}
        params = {
            'order': 'desc',
            'sort': 'votes',
            'site': 'stackoverflow',
            'pagesize': 3,
            'tagged': 'python;django',
            'intitle': intitle
        }

        r = requests.get(url, params=params, headers=headers)
        questions = r.json()

        print('')

        for question in questions['items']:
            print(question['title'])
            print(question['link'])

            print('')
```

尾部追加，保证异常发生时第一个执行

```python
# setting.py

if DEBUG:
    MIDDLEWARE += ['middleware.StackOverflowMiddleware', ]
```




参考：

[http://www.xuyasong.com/?p=927](http://www.xuyasong.com/?p=927)

[https://www.cnblogs.com/bainianminguo/p/9440610.html](https://www.cnblogs.com/bainianminguo/p/9440610.html)

[https://www.cnblogs.com/bainianminguo/p/9440610.html](https://www.cnblogs.com/bainianminguo/p/9440610.html)

[https://docs.djangoproject.com/en/2.2/topics/http/middleware](https://docs.djangoproject.com/en/2.2/topics/http/middleware/)
