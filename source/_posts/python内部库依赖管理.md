---
title: python内部库依赖管理
copyright: true
permalink: 1
top: 0
date: 2022-02-18 16:03:25
tags:
	- python
categories: python
password:
---

公司gitlab 上维护了一些Python内部库，最近出现了一些小毛病，解决的同时借此梳理一下。<!--more-->



## 内部库依赖正确写法



公司最近对gitlab 服务进行升级，由此导致了内部库下载失败。之前在requirements.txt 声明的链接如下：

![image-20220218162310393](/Users/tiger/own-project/DarrBlog/source/images/python内部库依赖管理/image-20220218162310393.png)

经排查发现gitlab 版本升级后，链接地址发生了变化，新的地址如下：

![image-20220218162718614](/Users/tiger/own-project/DarrBlog/source/images/python内部库依赖管理/image-20220218162718614.png)

了解后发现，其实 pip 就是将 gitlab 仓库的代码以压缩文件的形式下载下来并解压执行setup 生成python 运行环境的依赖库。

这么写还是直接获取tar 包的地址，适配性太差，不知道那天链接又改掉了，那么有什么好的方案吗

其实git 有提供获取依赖库的接口，通过调用以下链接实现依赖库的下载：

![image-20220218163758493](/Users/tiger/own-project/DarrBlog/source/images/python内部库依赖管理/image-20220218163758493.png)

## 版本管理

之前公司内部依赖库是拉取某一个分支，倘若采取这种方案会出现以下两个问题：

- 当底层依赖库升级后，上层所有库都需要做适配，否则会出现无法运行的情况
- 当部署否版本回退时，依赖无法回退至上一个版本



因此我们采取以下方案： 基于master 打tag 进行版本控制。当新的特性更新到master ,我们会对其进行打tag，声明版本。  如(v1.0.0),因此我们只需要在

业务库中修改依赖即可，如：

![image-20220221115331230](/Users/tiger/own-project/DarrBlog/source/images/python内部库依赖管理/image-20220221115331230.png)

而其他上层库则按需修改，也可以解决上面出现的问题。

## other

正常情况就没啥问题了，但是本人在测试环境进行部署时，发现情况并不乐观，执行以下命令(-q 控制台详细打印输出)：

![image-20220218171530761](/Users/tiger/own-project/DarrBlog/source/images/python内部库依赖管理/image-20220218171530761.png)

出现错误： `fatal: HTTP request failed`

查阅后发现，因为当前机器Centos 版本过低，只有Cento6.5，自带的的git 版本 1.7.1 ，并不支持此格式，因此需要对git 进行升级。

#### 升级git

1.安装依赖

``````shell
sudo yum install curl-devel expat-devel gettext-devel openssl-devel zlib-devel asciidoc
sudo yum install  gcc perl-ExtUtils-MakeMaker   
``````

2.卸载git

``````shell
sudo yum remove git
``````

3. 下载并安装高版本git

``````shell
wget https://github.com/git/git/archive/v2.2.1.tar.gz
tar zxvf v2.2.1.tar.gz
cd git-2.2.1
make prefix=/usr/local/git all
``````

4.添加至环境变量

因为是非root用户使用git，则需要配置下该用户下的环境变量：

``````shell
echo "export PATH=$PATH:/usr/local/git/bin" >> ~/.bashrc
source ~/.bashrc
``````

至此，git 升级完成，可以通过`git --version` 查看版本



再次执行下载命令，仍旧出现错误，`SSL connect error`，只是由于无法在服务器使用curl命令访问https域名,nss版本有点旧了，这该死的老机器😭：

![image-20220218164330516](/Users/tiger/own-project/DarrBlog/source/images/python内部库依赖管理/image-20220218164330516.png)

#### 更新nss

执行更新nss命令：

``````shell
yum update -y nss curl libcurl
``````

又又又出现问题啦😣`Cannot find a valid baseurl for repo: base`

需要更新yum 的下载源。。。。。。



#### 更换yum源

1. 备份原yum源

   ``````shell
   mv /etc/yum.repos.d/CentOS-Base.repo /etc/yum.repos.d/CentOS-Base.repo.backup
   ``````

   

2. 下载镜像源

   ``````shell
   #阿里云centos7的yum源：http://mirrors.aliyun.com/repo/Centos-7.repo
   #阿里云centos6的yum源：http://mirrors.aliyun.com/repo/Centos-6.repo
   wget -O /etc/yum.repos.d/Centos-Base.repo http://mirrors.aliyun.com/repo/Centos-6.repo
   ``````

捣鼓半天发现阿里源不在支持centos6 镜像源了，使用清华的吧。

执行以下命令替换镜像源：

``````shell
sed -i 's#vault.centos.org#mirrors.tuna.tsinghua.edu.cn/centos-vault#g' /etc/yum.repos.d/CentOS-Base.repo
``````

3.重新生成缓存

``````shell
#清除缓存
yum clean all
#重新生成缓存
yum makecache
``````

4. 更新nss

   ``````shell
   yum update -y nss curl libcurl
   ``````

5. 下载依赖

   ![image-20220218201207173](/Users/tiger/own-project/DarrBlog/source/images/python内部库依赖管理/image-20220218201207173.png)



参考：

[centos6使用git命令时fatal: HTTP request failed解决方法](https://blog.csdn.net/weixin_45588247/article/details/119887175?spm=1001.2101.3001.6661.1&utm_medium=distribute.pc_relevant_t0.none-task-blog-2%7Edefault%7ECTRLIST%7ERate-1.pc_relevant_paycolumn_v3&depth_1-utm_source=distribute.pc_relevant_t0.none-task-blog-2%7Edefault%7ECTRLIST%7ERate-1.pc_relevant_paycolumn_v3&utm_relevant_index=1)

[CentOS 6镜像源更换方法](https://support.huaweicloud.com/ecs_faq/ecs_faq_1009.html)

[git 使用报错SSL connect error](https://blog.csdn.net/erzhushashade/article/details/107360738?spm=1001.2101.3001.6650.2&utm_medium=distribute.pc_relevant.none-task-blog-2%7Edefault%7ECTRLIST%7ERate-2.pc_relevant_aa&depth_1-utm_source=distribute.pc_relevant.none-task-blog-2%7Edefault%7ECTRLIST%7ERate-2.pc_relevant_aa&utm_relevant_index=2)









