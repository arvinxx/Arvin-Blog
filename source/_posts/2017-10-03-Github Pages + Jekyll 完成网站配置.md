---

urlname: github-pages-and-jekyll
title: 博客建站记录（2）| Github Pages + Jekyll 完成网站配置
date: 2017-10-04
author: 空谷
header-img: http://pic.arvinx.com/pic/171016/11K6bKb14B.jpg
tags: 
  - Jekyll 
  - Github 
  - 博客 
  - Ruby
---


接下来开始写第二部分的内容，Github Pages 的创建与 Jekyll 的配置。

我是根据 [一步步在GitHub上创建博客主页][0] 这篇文章进行的博客搭建。

## GitHub Pages 创建
在 Github 上通过向导创建一个新的 repository，命名为 `arvinxx.github.io` 并保存。

这个时候访问 [arvinxx.github.io](https://arvinxx.github.io/) ，网站就显示出来了。
（可惜没有截图）

## 本地环境搭建

这一步不是必须的，不过还是建议做一下。因为在博客发布之前，通常都是需要在本地先检验一下的。但是实际上这个步骤其实让我有点困惑，所以在后续的配置过程中做了一些测试，才完全搞明白Jekyll 到底是怎么在 Github 上运行的。 
### Ruby安装
jekyll本身基于Ruby开发，因此，想要在本地构建一个测试环境，需要具有Ruby的开发和运行环境。由于我是第一次接触 Ruby 和 Ruby 的相关内容，所以一开始有点懵，因为主要参考文章中用的是 Windows，和我的系统不一致，就没有了参考性。后来我参考了 [Mac安装Ruby环境][1] 这篇文章，通过安装 RVM来控制 进行 Ruby 和 Gem 的管理与控制。

另外，原文最后的提到 `https://ruby.taobao.org`  这个网址也过期了，

> 这有可能是因为Ruby的默认源使用的是cocoapods.org，国内访问这个网址有时候会有问题，网上的一种解决方案是将远替换成淘宝的，替换方式如下：
>
> ``` bash
>  $ gem source -r https://rubygems.org/
>  $ gem source -a https://ruby.taobao.org
> ```
```

![](http://pics.arvinx.com/2017-10-04-15070601417555.jpg)

> #### RubyGems 镜像

> RubyGems 镜像的管理工作以后将交由 [Ruby China][3] 负责，以便能有更多的社区爱好者参与进来，保持持续发展。 
>
> 本站将不在继续维护，本站的维护者已经或即将参与到 [Ruby China 镜像][3] 的维护工作中，目前已将安装请求重定向到 [Ruby China 镜像][3]，请大家注意更换本地的 Gem Source。 
>
> ##### 如何使用？
> 
>``` bash
   $ gem sources --add https://gems.ruby-china.org/ --remove https://rubygems.org/
   $ gem sources -l
    *** CURRENT SOURCES ***    
    https://gems.ruby-china.org[C4D为什么能火？](media/C4D%E4%B8%BA%E4%BB%80%E4%B9%88%E8%83%BD%E7%81%AB%EF%BC%9F.md)
    # 请确保只有 gems.ruby-china.org
```


在完成 Ruby 环境配置后，又重新回到原教程，根据原教程的 `Gemfile和Bundle安装` 进行，本地端的服务就部署完成了。

![](http://pics.arvinx.com/2017-10-03-2017-10-04-1111.png)


然后是主题选择，作为一名全栈设计师，对颜值的要求自然是高的。最后选择了 [jekyll-theme-H2O][2] 这个主题。
![](http://pics.arvinx.com/2017-10-04-15070609383214.jpg)

在部署完本地服务器之后，又熟悉了一下 Jekyll 模板的结构

``` bash
	├── _config.yml # 配置文件
	├── _includes # 页面组件方便重用
	|   ├── footer.html # 页脚
	|   └── head.html # html文档的头部内容
	|   └── header.html # 顶部菜单栏
	|   └── pageNav.html # 文章列表分页组件
	├── _layouts # 布局模板
	|   ├── default.html # 默认模板
	|   └── post.html # 文章页面模板
	├── _posts # 这里放文章
	|   ├── 2017-05-03-elements-of-javascript-style.md # 命名格式：年-月-日-文章标题.md
	|   └── 2007-02-21-life-on-mars.md
	├── _site # Jekyll将源码处理后生成的站点文件，里面的内容可直接发布
	├── assets # 存放用于线上环境的静态资源，如需修改css和js文件请到dev文件夹
	|   ├── css # dev文件夹中sass编译后的样式文件
	|   └── fonts # 字体文件
	|   └── icons # 图标文件
	|   └── img #  图片文件
	|   └── js # dev文件夹中处理后的脚本文件
	├── dev # 开发文件
	|   ├── js # 存放脚本源码
	|   └── sass # 样式源码
	|       └── app.scss # 整合下面的所有样式文件
	|       └── base.scss # 引入字体、Reset部分样式
	|       └── common.scss # 模板的主要样式
	|       └── helper.scss # 工具样式
	|       └── layouts.scss # 响应式布局
	└── gulpfile.js # 自动化任务脚本
	└── index.html # 模板首页
	└── tags.html # 标签页面
	└── 404.html # 404页面
	└── package.json # 管理项目的依赖项
```

在这个过程中我产生了一个问题，Github 是拿 `_site` 里生成的文件来渲染网站，还是只是拿除 `_site` 文件夹外的模板渲染的网站？
经过测试后发现，没有 `_site` 文件夹内的文件，Github 照样能渲染网站。一方面说明 `_site` 是本地生成后文件，另一方面说明 Github 是基于Jekyll 模板进行网站渲染。
因此在 Push Repo 的时候可以把 `_site` 文件夹 ignore 掉。


## 参考资料

1. [一步步在GitHub上创建博客主页-最新版][0]
2. [Mac安装Ruby环境][1] 
3. [jekyll-theme-H2O][2]

[0]: http://www.pchou.info/ssgithubPage/2014-07-04-build-github-blog-page-08.html
[1]: http://www.jianshu.com/p/d22c1406ca33
[2]: https://github.com/kaeyleo/jekyll-theme-H2O
[3]:http://gems.ruby-china.org/







