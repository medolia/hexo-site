---
title: Github Pages 搭建个人博客网站
date: 2020/9/1 14:26:15
categories: 网站维护
tags: 
  - jekyll
  - markdown
  - github pages
excerpt: 简明教程，搭建一个基于 Jekyll 和 Github Pages 的个人博客网站
---



# 前言

现如今，个人博客网站有很多选择，可以选择依托于 [csdn](https://www.csdn.net/) 、[简书](https://www.jianshu.com/) 、[博客园](https://www.cnblogs.com/) 等知名大博客网站，一切的页面 UI 设计、字体大小、代码配色由主网站自动设定，作者只需要专注于博客内容即可。

如果对自定义设置这些页面元素有要求的话，可以选择较主流的基于 PHP(`best language ever!`)和 MYSQL 的 WordPress 或者基于 markdown 和 liquid 的 Jekyll 自主搭建，而最近在逛 Jekyll 模板库时看到了一个很中意的模板，想着实装的同时记录一下自己设置的过程作为教程给大家参考。

# 简要介绍

Jekyll [中文官网](http://jekyllcn.com/) [英文官网](https://jekyllrb.com/) 是一个简单静态博客网站生成器，而 Github Pages [官网](https://pages.github.com/) [官方文档](https://docs.github.com/en/github/working-with-github-pages) 依托于 Jekyll，博客作者只需要把 Jekyll 相关的文件上传至 Github 仓库，然后恰当设置即可生成一个响应式、满足大部分自定义需求的博客网站。

要使用 Jekyll，首先需要安装 Ruby [中文官网](https://www.ruby-lang.org/zh_cn/) 和 Gem [官网](https://rubygems.org/) ，然后使用 gem 安装 jekyll bundler `gem install jekyll bundler`。

接着使用 jekyll  new 命令新建一个博客网站 `jekyll new [site-name]` 

`cd [site-name]` 跳转至存有博客源文件的文件夹，`bundle install` 下载相关依赖，最后 `bundle exec jekyll serve` 开启服务。现在就可以在浏览器中输入 `http://localhost:4000/` 访问博客网站了：

![截屏2020-09-01 15.53.10](https://tva1.sinaimg.cn/large/007S8ZIlgy1gib7bognrlj312q0qi12q.jpg)

![截屏2020-09-01 15.56.23](https://tva1.sinaimg.cn/large/007S8ZIlgy1gib7f1ccdrj314y0u04hg.jpg)

打开网站对应的文件夹，可以看到 jekyll 默认的项目结构：

![截屏2020-09-01 16.06.47](https://tva1.sinaimg.cn/large/007S8ZIlgy1gib7purhfsj30ke0sy41i.jpg)

挑重点说明一下，**_posts** 存放所有 markdown 编写的博客文章，文件名为 **4 位数年份-2 位月份-2 位日-博客标题**；**feed.xml** 是自动生成的 RSS feed；**_config.yml** 是主要的配置文件，可以看到里面指定了默认的 minima 主题，使用了 feed 生成插件 **jekyll-feed**，以及博客标题、作者邮箱、网站描述等等；**Gemfile** 指定 gem 依赖，可以设定博客使用 Github Pages 支持的最新版本插件防止发生文件解析错误。 

# 使用模板

Jekyll 有很多对应模板网站，比如：

+ <https://jekyllthemes.io>

+ <http://jekyllthemes.org>

+ <https://jekyll-themes.com>

+ <https://jamstackthemes.dev/ssg/jekyll> 

里面有些模板需要付费，不过大部分模板是开源的，也非常好看和有创意，随意选择一个自己喜欢的然后下载压缩包即可，然后参照作者提供的用户 API 文档自定义页面、微调页面元素，比如我下载的是一个名为 **minimal-mistakes** 的主题，参考它的 [设置文档](https://mmistakes.github.io/minimal-mistakes/) 作了些自己的微调得到可以上传于 Github Pages 的 [成品](https://github.com/medolia/medolia.github.io) 。

# 上传至 Github

首先新建一个名字为 **{username}.github.io** 的仓库，Github 会自动解析仓库为 Github Pages 文件，具体可以在仓库的 **settings** 里确认：

![截屏2020-09-01 16.39.01](https://tva1.sinaimg.cn/large/007S8ZIlgy1gib8ndzi63j318q0u07ax.jpg)

值得一提的是这里的 **Theme Chooser** 可以帮选择困难症快速配置主题，**Custom domain** 栏可以自定义域名。

然后使用 git 链接远程仓库，具体教程可以参考[这篇文章](https://www.jianshu.com/p/1b65ed31da97) 。

现在，任何设备都可以通过浏览器访问 **{username}.github.io** 看到你的博客：

![截屏2020-09-01 16.44.11](https://tva1.sinaimg.cn/large/007S8ZIlgy1gib8srwehjj314y0u0hdy.jpg)

# 配置域名

很遗憾，默认状态百度等搜索引擎并不支持爬取 Github Pages 的内容，我们还需要额外购买一个自己的专属域名，阿里云、百度熊掌号、花生壳域名等都可以，不同网站配置不同，这里就不赘述了。

总之，激活 Github Pages 并配置好域名后，一个酷酷的自定义博客网站就能被大家看到了，开始向世界表达你的想法吧。

# One More Thing

markdown 语法说明: [markdown-it](https://markdown-it.github.io/)

markdown 软件推荐（mac os）：

+ Typora: 免费，所见即所得，支持匿名上传图片至微博（无需注册）。
+ Mweb: 付费，强大，好评如潮，用过就知道。

我的博客链接: [Medolia 个人博客](https://medolia.github.io/)

