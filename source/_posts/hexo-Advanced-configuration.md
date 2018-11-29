---
title: hexo blog高级配置
date: 2018-11-28 10:53:42
categories:
copyright: true
tags: blog
---
通过hexo创建完简单blog，是不是还觉得缺点啥呢？那就让我们进入高级配置把。
<!--more-->

本blog使用主题为[NexT6.5.0](https://github.com/theme-next/hexo-theme-next)会和[iissnan](https://github.com/iissnan/hexo-theme-next)的版本有些许不一样。

如果不会搭建blog,可以参考uchuhimo老哥的文章[如何使用 Hexo 和 GitHub Pages 搭建这个博客](https://uchuhimo.me/2017/04/11/genesis/)，比较详细，我就不重复造轮子了。

#### 自定义文章的默认头部信息
创建新的页面`hexo new "name"`
查看`source\_posts\name.md`：
```cs
---
title: name
date: 2018-11-28 10:53:42 
tags: 
---
```
添加`scaffolds/post.md`：
```cs
---
title: {{ title }}
date: {{ date }}
tags:               #标签
categories:         #分类
copyright: true     #版权声明
permalink: 01       #文章链接
top: 0              #置顶优先级
password:           #密码保护
---
```

#### 文章代码样式修改
- 行内代码
修改`themes\next-reloaded\source\css\_custom\custom.styl`文件：
``` css
code {
    color: #c7254e;
    background: #f9f2f4;
    border: 1px solid #d6d6d6;
    padding:1px 4px;
    word-break: break-all;
    border-radius:4px;
}
```
- 区块代码
修改`themes\next-reloaded\config.yml`文件：
```
    # 样式种类： normal | night | night eighties | night blue | night bright
    highlight_theme: night
```

#### 文章结束语设置
- 添加template
`themes\next-reloaded\layout\_macro`中新建 `passage-end-tag.swig`,内容如下：
```html
<div>
    {% if not is_index %}
        <div style="text-align:center;color: #ccc;font-size:14px;">
              -------------本文结束  感谢您的阅读-------------
        </div>
    {% endif %}
</div>
```
- 导入template
在`themes\next-reloaded\layout\_macro\post.swig`文件中,添加
```html
<div>
  {% if not is_index %}
      {% include 'passage-end-tag.swig' %}
  {% endif %}
</div>
{#####################}
{### END POST BODY ###}
{#####################}
```
- 配置
在`themes\next-reloaded\_config.yml`添加：
```cs
passage_end_tag:
    enabled: true
```

#### 文章底部标签修改
修改`themes\next-reloaded\layout\_macro\post.swig`,search`rel="tag">#`,将 `#`更改为`<i class="fa fa-tag"></i>`

#### 搜索功能
- 安装
```
npm install hexo-generator-searchdb --save
```
- 配置
   修改`themes\next-reloaded\_config.yml`:
```cs
local_search:
  enable: true
  trigger: auto
  top_n_per_article: 1
```

   在`_config.yml`中，添加：
```cs
search:
  path: search.xml
  field: post
  format: html
  limit: 10000
```

#### 连接样式美化
添加`themes\next-reloaded\source\css\_custom\custom.styl`:
```css
.post-body a{
  color: #0593d3;
  border-bottom: none;
  border-bottom: 1px solid #0593d3;
  &:hover {
    color: #fc6423;
    border-bottom: none;
    border-bottom: 1px solid #fc6423;
  }
}
```

#### 文章添加阴影效果
添加`themes\next-reloaded\source\css\_custom\custom.styl`:
```css
.post {
   margin-top: 60px;
   margin-bottom: 60px;
   padding: 25px;
   -webkit-box-shadow: 0 0 5px rgba(202, 203, 203, .5);
   -moz-box-shadow: 0 0 5px rgba(202, 203, 204, .5);
  }
```

#### 添加动态背景
打开`\themes\next-reloaded\layout\_layout.swig`，在`< /body>`前添加代码：
```html
{% if theme.canvas_nest %}
<script type="text/javascript" src="//cdn.bootcss.com/canvas-nest.js/1.0.0/canvas-nest.min.js"></script>
{% endif %}
```
添加`themes\next-reloaded\_config.yml`:
```cs
# --------------------------------------------------------------
# background settings
# --------------------------------------------------------------
# add canvas-nest effect
# see detail from https://github.com/hustcc/canvas-nest.js
canvas_nest: true
```
    
    配置项说明
    color ：线条颜色, 默认: '0,0,0'；三个数字分别为(R,G,B)
    opacity: 线条透明度（0~1）, 默认: 0.5
    count: 线条的总数量, 默认: 150
    zIndex: 背景的z-index属性，css属性用于控制所在层的位置, 默认: -1

#### 添加百度分享功能
添加`themes\next-reloaded\_config.yml`:
```cs
baidushare:
  type: slide
  baidushare: true
```

#### 小萌物
- install
推荐使用cnpm
`npm install --save hexo-helper-live2d`
- 选择小萌物
可以查看[hexo-helper-live2d](https://github.com/EYHN/hexo-helper-live2d)
```
live2d-widget-model-chitose
live2d-widget-model-epsilon2_1
live2d-widget-model-gf
live2d-widget-model-haru/01 (use npm install --save live2d-widget-model-haru)
live2d-widget-model-haru/02 (use npm install --save live2d-widget-model-haru)
live2d-widget-model-haruto
live2d-widget-model-hibiki
live2d-widget-model-hijiki
live2d-widget-model-izumi
live2d-widget-model-koharu
live2d-widget-model-miku
live2d-widget-model-ni-j
live2d-widget-model-nico
live2d-widget-model-nietzsche
live2d-widget-model-nipsilon
live2d-widget-model-nito
live2d-widget-model-shizuku
live2d-widget-model-tororo
live2d-widget-model-tsumiki
live2d-widget-model-unitychan
live2d-widget-model-wanko
live2d-widget-model-z16
```

找到自己心仪的install。 example：`npm install live2d-widget-model-haruto`

- 配置
```cs
# 宠物
live2d:
  enable: true
  scriptFrom: local
  pluginRootPath: live2dw/
  pluginJsPath: lib/
  pluginModelPath: assets/
  tagMode: false
  log: false
  model:
    use: live2d-widget-model-haruto
  display:
    position: right
    width: 150
    height: 300
  mobile:
    show: true
```

#### 添加版权信息
- 添加格式
在`themes\next-reloaded\layout\_macro\post.swig`,`post-footer `所在的标签下，添加以下内容：
```html
<footer class="post-footer"> 原有内容
<div>    
 {# 此处判断是否在索引列表中 #}
 {% if not is_index %}
<ul class="post-copyright">
  <li class="post-copyright-author">
      <strong>本文作者：</strong>{{ theme.author }}
  </li>
  <li class="post-copyright-link">
    <strong>本文链接：</strong>
    <a href="{{ url_for(page.path) }}" title="{{ page.title }}">{{ page.path }}</a>
  </li>
  <li class="post-copyright-license">
    <strong>版权： </strong>
    本站文章均采用 <a href="http://creativecommons.org/licenses/by-nc-sa/3.0/cn/" rel="external nofollow" target="_blank">CC BY-NC-SA 3.0 CN</a> 许可协议，请勿用于商业，转载注明出处！
  </li>
</ul>
{% endif %}
</div>
```
- 显示格式
添加`themes\next-reloaded\source\css\_custom\custom.styl`:
```css
.post-copyright {
    margin: 1em 0 0;
    padding: 0.5em 1em;
    border-left: 3px solid #ff1700;
    background-color: #f9f9f9;
    list-style: none;
}
```