---
title: annual_summary
copyright: true
permalink: 1
top: 0
date: 2020-01-15 10:30:44
tags: 
categories: summary
password:
---

2019已悄然离去，回顾这一整年，思考良多。2019年也是正式步入社会的一年，在技术领域，在团队协作方面都有很多提升，话不多说，切入正题。<!--more-->

2018年下半年以小萌新的姿态从校园步入社会，因公司技术栈需求，从Java 转向 python,并逐渐迷恋这款方便快捷，语义清晰的语言。
在2019年，随着python语言的深入，也开展了很多相关工作。起初接触了公司api开发，对django有了一定了解。
随后转向业务开发，受迫于初创公司设计文档，注释严重匮乏，逻辑复杂且编码风格层出不穷，吃劲苦口，
也深刻意识到python这种动态的强类型解释语言在团队开发中，在缺少语言本生强校验的前提下，
更需要制定一些措施来避免编写这类动态语言代码重构火葬场的现象，因此在个人日常编码过程中尽可能详尽的提供描述信息，
并配合 Type Hint 做到尽可能减少团队开发中不必要的其他因素的负担。

公司项目在这一年里也经历了逐步演化的过程，我有幸参与其中，并主导了部分演进工作，受益颇多。
起初后端业务相关的项目server使用 HTTPServer (python原生支持的方式),过于底层且功能单一，绝大多数功能都需要自己逐一实现，
如需要满足高并发的场景还需要遵守WSGI APPLICATION 规范来搭配现在主流的WSGI SERVER，等等一系列因素使得维护成本增高，
且吃力不讨好(python社区 明明就对 app 和 sever 进行拆分，让程序员专注解决自己的业务需求)，我们团队便着手筹划项目演进。
现在主流的web server 主要分为两大类:同步框架和异步框架。当时主要考虑因素在不影响现有架构的前提下保证轻量且高并发。
显然异步框架更能满足高并发需求，最终选择aiohttp作为服务器框架。接触到异步编程，对这个邻域有了更深的理解。
但在当前环境下，其实异步框架生态还并不友好，底层支持也不太完善，而且和之前同步模式差异过大。导致在迁移过程过程繁琐。

随后老板便指派我调研并主导架构的演进，我参考框架和现有项目的适配程度和框架生态完善程度，
结合主流架构设计思路，最终选择了 flask + redis + gunicorn + nginx + docker 的架构。
这个过程接触到了很多之前尚未接触或略有了解的知识领域，也意识到了自己知识的浅显，求知的道路还很漫长。
在此期间，还主导搭建了一个前端项目，采用 vue + iview + nginx + docker 的架构。得意于架构的精良设计和前端生态的不断完善，
让我一个不怎么会写css html 的新手也能信手捏来。

当然，在团队协作上也有了很大的提升，在讨论问题的时候最好预先提供上下文，再提出问题的时候最好有深度的思考，在解决问题的时候也要有更高的扩展性。
对任务要有优先级的划分，不至于手足无措，对大型任务要设定排期，保证工期不被延误。

总的来说，2019是转变的一年，从小萌新初入职场，到老油条车臣沙场。以上就是我对自己的一个总结。2020，fighting!!!






