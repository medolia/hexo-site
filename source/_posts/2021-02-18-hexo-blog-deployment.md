---
title: hexo 使用主题时搭建博客部署
date: 2021/2/18 19:17:15
categories: 网站维护
tags:
  - hexo
  - git
excerpt: 前不久自己学着做了个 hexo 博客网站，由于之前已经做过 jekyll 版本的，想必肯定是触类旁通、信手拈来。安装 node.js、npm 依赖，下好主题并按照作者的 doc 文档修改好主题配置，美滋滋地按教程部署，然而。。。
---

## 问题展示

访问 github pages 的绑定域名后，可以看到主题相关文件好像没有完全载入，只有极少数 js 和 html 文本显示
![错误部署的博客页面](https://medoliablog.oss-cn-hangzhou.aliyuncs.com/2021/02/18/16136523533213.jpg)

## 问题解析

在这之前，需要比较下 jekyll 和 hexo 两种博客在含主题后源码的组织方法。

首先是 jekyll

![jekyll 项目组织](https://medoliablog.oss-cn-hangzhou.aliyuncs.com/2021/02/18/16136526378621.jpg)

可以看到，主题并没有单独存放，而是直接与主项目耦合在了一起，这样的好处是部署省心，而且 github pages 会自动解析项目文件，这样就不需要像 hexo 一样需要单独创建一个 branch 存放已转为 html 的博客页面。

再来看 hexo 

![](https://medoliablog.oss-cn-hangzhou.aliyuncs.com/2021/02/18/16136528883140.jpg)

可以看到有单独的 themes 文件夹存放主题文件，主项目和主题文件都存在 _config.yml 配置文件（相同配置项下前者的优先级更高）。这样做的好处是可以将独立的第三方主题 fork 进搭建好的项目里，用户可以把用户信息放在主项目里，主题的配置文件只用于定制主题的自定义配置。

这几乎完美契合 git [**子模块**](https://git-scm.com/book/zh/v2/Git-%E5%B7%A5%E5%85%B7-%E5%AD%90%E6%A8%A1%E5%9D%97) 功能的应用需求，事实上 hexo 也是这么做的，所有的第三方主题都作为独立的子项目。而这样做的坏处是如果没有指定子模块的从属关系（即创建 .gitmodules 文件），直接使用部署功能的话会导致主题文件无法加载（因为找不到主题文件）。

## 解决方法

这里给出一种比较建议的搭建 hexo 博客的流程。

1. `git clone` 或者 `hexo init` 得到博客的主项目，配置 **_config.yml** 中的 site（本人相关信息）、deploy（部署相关信息）、url（资源格式相关）等选项。
2. `npm install` `rm -rf themes/<themeName>` `git rm -r --cached themes/<themeName>` 安装依赖后，清空主题文件夹
3. `git submodule add https://github.com/<username>/<repoName>.git themes/<themeName>` 将 fork 好的主题源码地址为参添加子模块
4. 配置主题的 **_config.yml** 的自定义选项（根据作者给出的 user guide 文档）
5. `cd themes/<themeName>` `git add .` `git commit -m "init"` `git push -u origin master` 提交子模块的更新
6. 主项目 `hexo deploy` 完成部署。

## 外链

[Hexo 官方中文部署指导文档](https://hexo.io/zh-cn/docs/one-command-deployment.html)

[git 子模块介绍](https://git-scm.com/book/zh/v2/Git-%E5%B7%A5%E5%85%B7-%E5%AD%90%E6%A8%A1%E5%9D%97)

[子模块解决 Hexo 第三方主题同步更新问题](https://www.dazhuanlan.com/2020/02/02/5e369bc4267b3/)


