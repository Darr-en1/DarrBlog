---
title: 'ChatGPT很牛,我很慌'
copyright: true
permalink: 1
top: 0
date: 2023-01-05 10:24:55
tags:
    - ChatGPT
categories:
    - ChatGPT
password:
---
ChatGPT最近实在太火了，甚至很多圈外的小伙伴都知道了。它非常人性化的对话模型和强大的上下文关联,甚至让我意识不到我是在同一个机器人对话，它甚至可以协助编程，让我一度怀疑可能还没等到35岁毕业就要面临被AI淘汰的尴尬囧境。ChatGPT真的神乎其神，可以取代现有岗位吗? 知己知彼，方能百战不殆，赶紧来认识认识这个强大对手。<!--more-->



## ChatGPT是什么



ChatGPT是一种自然语言生成模型，它基于GPT-3构建，用于在自然对话中进行语言生成。它具有自然语言理解和语言生成的能力，能够以人类一样的方式回答问题并发起对话。 ChatGPT是一个高效的工具，可以在构建聊天机器人、智能助手、问答系统等应用程序时使用。



以上是ChatGPT自己的回答。简单点讲，ChatGPT就是一个聊天机器人，但是它非常智能，能够关联上下文，参与自然对话，并理解他们的意图，从而使对话更加流畅自然。



以下是我喜欢的一个up主对于ChatGPT的介绍，有涉及基本原理，可以方便大家理解。

<iframe src="//player.bilibili.com/player.html?aid=776432720&bvid=BV1Q14y1N7mU&cid=927246944&page=1" scrolling="no" border="0" frameborder="no" framespacing="0" allowfullscreen="true"> </iframe>

## ChatGPT插件推荐

由于ChatGPT的异常火爆，社区基于ChatGPT的应用多如雨后春笋，在此我就推荐几个有意思的插件。但前提是需要拥有一个OpenAI的账号，OpenAI没有中国开放注册，注册一般都会提示OpenAI's services are not available in your country。

### 前期准备

- 安全上网，最好是美国IP，香港没用，需开启全局代理
- 外国邮箱
- 有一个能收到验证码的外国手机号码，可以前往 https://sms-activate.org/cn 购买一个虚拟账号用于接验证码，注意这是收费的，但也很便宜，虚拟账号充0.2美元就足够购买一次印度号码

如果开启代理后还会出现 OpenAI's services are not available in your country

在网页的搜索栏输入`javascript:window.localStorage.removeItem(Object.keys(window.localStorage).find(i=>i.startsWith('@@auth0spajs'))) `然后刷新即可

注⚠️： javascript: 要手动输入 无法复制

### 推荐

- 搜索引擎插件：https://github.com/wong2/chat-gpt-google-extension

当你在搜索引擎搜索时，屏幕的右侧则会显示ChatGPT对于这个搜索问题的回答，你可以同搜索引擎进行对比，从而找到更合适的答案。

- 文章摘要生成插件：https://github.com/clmnin/summarize.site

插件会对文章进行简要概括，方面你快速了解文章主要内容

- 代码错误解释插件：https://github.com/shobrook/stackexplain

使用 ChatGPT 解释代码的错误消息，便于问题排查

## 个人微信接入ChatGPT

最后，实现个人微信接入 ChatGPT

### 接入前提

登录 openai：[beta.openai.com/login/](https://link.juejin.cn/?target=https%3A%2F%2Fbeta.openai.com%2Flogin%2F)，然后点击页面右上角的头像，进入 View API keys，

创建一个新的秘钥，用于和 openai 认证和交互。

### 安装部署

下载源代码并下载依赖

```go
git clone https://github.com/djun/wechatbot.git
cd wechatbot 
go mod tidy
```

添加软连接并，修改配置

```she
ln -s config.dev.json config.json
vim config.dev.json
```

将刚刚生成的秘钥写入到config.dev.json 的`api_key`中

![image-20230109221626880](/images/ChatGPT很牛-我很慌/image-20230109221626880.png)

运行主程序

```go
go run main.go
```

这时会页面会弹出二维码

![image-20230109222245421](/images/ChatGPT很牛-我很慌/image-20230109222245421.png)

扫码后你的微信就变成了ChatGPT了。

这是最终的效果，单聊直接发信息即可接收到ChatGPT的回复

![image-20230109222535420](/images/ChatGPT很牛-我很慌/image-20230109222535420.png)

## 对于ChatGPT的思考

ChatGPT 确实很厉害，它可以写邮件、稿子、编写简单代码、参与内容创作等等，相信在未来人工智能肯定会替代一部分简单重复的工作，这也是科技进步的结果。

那我们应该担心吗？大可不必。人类之所以可以在食物链的顶端，因为我们拥有智慧以及善于学习，并具备创造力和想象力。AI替代的是能力，汽车之所以更够替代马车，是因为它跑得快还不停歇。AI的存在也必然是推动人类的进步。因此我们应该保持开放，不断提升自己的核心竞争力，让这些工具更好的服务于我们。

工具不会替代人，当然，前提你不是一个工具人。



参考

https://readdevdocs.com/blog/makemoney/%E4%B8%AD%E5%9B%BD%E5%8C%BA%E6%B3%A8%E5%86%8COpenAI%E8%B4%A6%E5%8F%B7%E8%AF%95%E7%94%A8ChatGPT%E6%8C%87%E5%8D%97.html#%E5%89%8D%E6%9C%9F%E5%87%86%E5%A4%87