---
title: mac升级踩坑实录
copyright: true
permalink: 1
top: 0
date: 2022-04-15 15:48:22
tags:
    - 杂谈
    - mac
    - Monterey
    - python
categories:
    - 杂谈
password:


---
昨天在家看视频，嫌电脑屏幕太小，奈何没有显示屏，于是拿出自己的ipad当作副屏，可怎么都连不上，emo中下意识看到mac 的系统偏好设置里提醒我mac升级，我个人对于软件版本升级一直是非常热衷的，对新事物的接受程度也是非常高的，于是乎就将mac 进行了版本升级，但但但是呢？还真发现了一堆坑，老泪纵横呀，希望大家也能引以为戒吧！😭😭 <!--more-->



以下是升级后的机器概览

![image-20220415155810754](/images/mac升级踩坑实录/image-20220415155810754.png)

### Cannot Run GIt

MacOS升级后，我打开goland 发现 git 的文件颜色标记消失，同时右下角出现错误弹框：

![IntelliJ IDEA Cannot run git.png](/images/mac升级踩坑实录/git-ae4bfb0b8cf241618d69662bd8dcd7f3.png)

在控制台输入`git --version`，也是报错：

``````bash
xcrun: error: invalid active developer path (/Library/Developer/CommandLineTools), missing xcrun at: /Library/Developer/CommandLineTools/usr/bin/xcrun
``````

#### 解决

```shell
xcode-select --install
```

此时会出现弹框

![image-20220415161413336](/images/mac升级踩坑实录/image-20220415161413336.png)

点击“安装”，根据提示安装即可。

```shell
git --version
git version 2.30.1 (Apple Git-130)
```



打开goland,还是不行呀，怎么办呢？

重新配置一下git 吧

```shell
 which git
/usr/bin/git
```

对goland配置

![image-20220415161840883](/images/mac升级踩坑实录/image-20220415161840883.png)

重启一下goland,ok了，终于又可以gggggit 了。



### /usr/bin/python: bad interpreter

公司跳板机每次登陆都需要使用动态密钥，通过宁盾令牌配置，连接变得非常繁琐，因此我通过使用ITerm profiles 配置脚本实现了自动获取密钥一键登录，但是升级后使用不了，出现以下错误

![image-20220415162049419](/images/mac升级踩坑实录/image-20220415162049419.png)



我第一步去看看 /usr/bin/python 是否存在,发现果然不存在了，输入python失败

![image-20220415162639550](/images/mac升级踩坑实录/image-20220415162639550.png)

只能输入python3，python2 也失效了，怎么回事呢？

![image-20220415162811608](/images/mac升级踩坑实录/image-20220415162811608.png)

在[macOS Monterey 12.3 的 beta 发行说明中](https://developer.apple.com/documentation/macos-release-notes/macos-12_3-release-notes)，Apple 声明目录 /usr/bin/Python 将被完全删除。

天坑呀，天坑呀

![image-20220415163347285](/images/mac升级踩坑实录/image-20220415163347285.png)brew 并未找到 python2的包，看来python2是彻底被mac放弃了😭 

不过，想想也很正常，python3 2008问世，但是python2，python3 在很长一段时间共存， 社区一直饱受两个大版本的摧残，官方正式宣布，Python2将于**2020年1月1日**停止更新和维护。人总要向前看的，python也在向更好的方向发展，包袱是该卸一卸了，短期阵痛才能着急更长的机遇。

Python 是我工作后一直使用的语言，虽然存在一些问题，得益于它的灵活，便捷，一直是我非常喜欢的一门语言。也希望整个社区欣欣向荣，蓬勃发展。



人生苦短，我用python。 

respect！！！！！