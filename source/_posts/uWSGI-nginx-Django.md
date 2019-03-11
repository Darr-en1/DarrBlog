---
title: uWSGI_nginx_Django
copyright: true
permalink: 1
top: 0
date: 2019-03-11 18:34:09
tags:
    - django
    - uWSGI
    - nginx
    - linux 
categories: django
password:
---


通过uWSGI+nginx+Django搭建生产web服务<!--more-->


### 系统环境参数

操作系统：Ubuntu 18.04.1 LTS

python版本：3.6.7

Django版本：2.1.7

nginx版本：nginx/1.14.0 (Ubuntu)



### 服务搭建

本文使用虚拟环境保证环境纯净

```
virtualenv -p /usr/bin/python3 uwsgi-tutorial
cd uwsgi-tutorial
source bin/activate
pip install uwsgi
pip install django

```
退出当前虚拟环境`deactivate`

#### uWSGI介绍
一个web服务器面对的是外部世界。它能直接从文件系统提供文件 (HTML, 图像， CSS等等)。然而，它无法 *直接*与Django应用通信；它需要借助一些工具的帮助，这些东西会运行运用，接收来自web客户端（例如浏览器）的请求，然后返回响应。

一个Web服务器网关接口（Web Server Gateway Interface） - WSGI - 就是干这活的。 WSGI 是一种Python标准。

uWSGI是一种WSGI实现。在这个教程中，我们将设置uWSGI，让它创建一个Unix socket，并且通过WSGI协议提供响应到web服务器。

##### 基础测试
当前目录 `/home/darren`

创建`test.py` `vim test.py`

```python
def application(env, start_response):
    start_response('200 OK', [('Content-Type','text/html')])
    return [b"Hello World"] # python3
    #return ["Hello World"] # python2
```

`uwsgi --http :8000 --wsgi-file test.py`

- http :8000: 使用http协议，端口8000
- wsgi-file test.py: 加载指定的文件，test.py

浏览器访问`http://192.168.98.129:8000/` 显示Hello World

组件栈
```
the web client <-> uWSGI <-> Python
```

##### 运行Django

*django基础运行方式*

```
django-admin.py startproject mysite
cd mysite
vim mysite/setting.py
# 设置允许访问
ALLOWED_HOSTS = ['*']

python manage.py runserver
```
*uwsgi运行Django*

```
cd mysite
uwsgi --http :8000 --module mysite.wsgi
```

组件栈
```
the web client <-> uWSGI <-> Django
```

#### nginx
浏览器一般不会直接和uWSGI打交道，它通过连接web服务器，利用uWSGI创建Unix socket，和Django应用建立关系。

```
# install
sudo apt-get install nginx

# basic operation
sudo /etc/init.d/nginx start
sudo /etc/init.d/nginx stop
sudo /etc/init.d/nginx restart
```

##### 为站点配置nginx

需要 `uwsgi_params`文件
```
uwsgi_param  QUERY_STRING       $query_string;
uwsgi_param  REQUEST_METHOD     $request_method;
uwsgi_param  CONTENT_TYPE       $content_type;
uwsgi_param  CONTENT_LENGTH     $content_length;

uwsgi_param  REQUEST_URI        $request_uri;
uwsgi_param  PATH_INFO          $document_uri;
uwsgi_param  DOCUMENT_ROOT      $document_root;
uwsgi_param  SERVER_PROTOCOL    $server_protocol;
uwsgi_param  REQUEST_SCHEME     $scheme;
uwsgi_param  HTTPS              $https if_not_empty;

uwsgi_param  REMOTE_ADDR        $remote_addr;
uwsgi_param  REMOTE_PORT        $remote_port;
uwsgi_param  SERVER_PORT        $server_port;
uwsgi_param  SERVER_NAME        $server_name;
```


`vim mysite_nginx.conf`


通过配置告诉nginx提供来自文件系统的媒体和静态文件，以及处理那些需要Django干预的请求
```
# mysite_nginx.conf
# the upstream component nginx needs to connect to
upstream django {
    # server unix:///home/darren/mysite/mysite.sock; # for a file socket
    server 127.0.0.1:8001; # for a web port socket (we'll use this first)
}

# configuration of the server
server {
    # the port your site will be served on
    listen      8000;
    # the domain name it will serve for
    server_name .example.com; # substitute your machine's IP address or FQDN
    charset     utf-8;

    # max upload size
    client_max_body_size 75M;   # adjust to taste

    # Django media
    location /media  {
        alias /home/darren/mysite/media;  # your Django project's media files - amend as required
   }

    location /static {
        alias /home/darren/mysite/static; # your Django project's static files - amend as required
    }

    # Finally, send all non-media requests to the Django server.
    location / {
        uwsgi_pass  django;
        include     /home/darren/mysite/uwsgi_params; # the uwsgi_params file you installed
    }
}

```

设置链接让nginx可以读取
`sudo ln -s /home/darren/mysite/mysite_nginx.conf /etc/nginx/sites-enabled/`

##### 部署静态文件
```
vim mysite.setting.py
STATIC_ROOT = os.path.join(BASE_DIR, "static/")
```
运行

`python manage.py collectstatic`

重启nginx

`sudo /etc/init.d/nginx restart`


#### uwsgi + nginx +Django

##### TCP端口socket

使用uwsgi协议，端口为8001,nginx配置中暴露8000端口与uWSGI通信
`uwsgi --socket :8001 --module mysite.wsgi`

##### 使用Unix socket

编辑 `mysite_nginx.conf`:

```
    server unix:///home/darren/mysite/mysite.sock; # for a file socket
    # server 127.0.0.1:8001; # for a web port socket (we'll use this first)
```

运行
`uwsgi --socket mysite.sock --module mysite.wsgi`

可能会出现启动失败，可以 `tail -f /var/log/nginx/error.log`动态查看错误信息。

本人在操作的时候出现的错误
``` shell
2019/03/11 07:18:51 [crit] 51025#51025: *55 connect() to unix:///home/darren/mysite/mysite.sock failed (13: Permission denied) while connecting to upstream, client: 192.168.98.1, server: example.com, request: "GET / HTTP/1.1", upstream: "uwsgi://unix:///home/darren/mysite/mysite.sock:", host: "192.168.98.129:8000"
```

更改套接字文件的权限，以便www-data可以写入它。

`uwsgi --socket mysite.sock --module mysite.wsgi --chmod-socket=666`

组件栈

`the web client <-> the web server <-> the socket <-> uWSGI <-> Django`


##### 配置uWSGI以允许.ini文件

命令行启动参数过多可以通过启动配置文件的方式

创建`vim mysite_uwsgi.ini`

```shell
# mysite_uwsgi.ini file
[uwsgi]

# Django-related settings
# the base directory (full path)
chdir           = /home/darren/mysite
# Django's wsgi file
module          = mysite.wsgi
# the virtualenv (full path)
home            = /home/darren/uwsgi-tutorial
# process-related settings
# master
master          = true
# maximum number of worker processes
processes       = 2
# the socket (use the full path to be safe
socket          = /home/darren/mysite/mysite.sock
# ... with appropriate permissions - may be needed
chmod-socket    = 666
# clear environment on exit
vacuum          = true

```
运行配置文件启动服务

`uwsgi --ini mysite_uwsgi.ini`


本人在操作时出现错误，如下：

```
no request plugin is loaded, you will not be able to manage requests. you may need to install the package for your language of choice, or simply load it with --plugin.
```

在[stackoverflow](https://stackoverflow.com/questions/10748108/nginx-uwsgi-unavailable-modifier-requested-0)找到解决方案:

```
apt-get install uwsgi-plugin-python3
```
在mysite_uwsgi.ini添加
```
plugins = python3
```

#### 探讨

##### Django 自己就可以搭建web服务,为什么用uWsgi?

Django自带wsgi服务器，但是性能不好，单进程。uWSGI支持的并发量更高
方便管理多进程，发挥多核的优势提升性能。

##### uWSGI可以作为Django web服务端，为什么还需要用到nginx?
uWSGI，是一个程序，充当Web服务器或中间件。如果架构是Nginx+uWSGI+APP，uWSGI是一个中间件.如果架构是uWSGI+APP，uWSGI是一个服务器。

- nginx改进了静态资源的处理，可以显着减少服务器和网络负载
- nginx也可以缓存一些动态内容
- nginx 作为专业服务器，暴露在公网相对比较安全
- nginx可以进行多台机器的负载均衡
- nginx可以更好地配合CDN


参考：

[uwsgi](https://uwsgi-docs-zh.readthedocs.io/zh_CN/latest/tutorials/Django_and_nginx.html)

[Nginx+uWSGI+Django原理](https://www.cnblogs.com/Xjng/p/aa4dd23918359c6414d54e4b972e9081.html)

[stackExchage](https://serverfault.com/questions/590819/why-do-i-need-nginx-when-i-have-uwsgi)