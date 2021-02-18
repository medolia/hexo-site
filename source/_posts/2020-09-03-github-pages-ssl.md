---
title: Github Pages 绑定域名并启用 SSL 证书
date: 2020/9/3 14:26:15
categories: 网站维护
tags:
  - github pages
  - SSL
  - 域名
excerpt: 完成 Github Pages 绑定自定义域名相关配置
---

# 前言

正常发表 Github Pages 后，Github 官方会使用自己的 DNS 服务器并配套提供 SSL 证书，这样虽然所有人都能访问且网站也处于安全防护中。但要想自己的博客内容被国内的搜索引擎识别并爬取则需要购买域名、配置 SSL、网站公安备案等一系列较（fan）复（dao）杂（zha）的操作，这篇文章以已有发表的 GIthub Pages 为前提介绍如何完成上述配置。

# 购买域名并配置

国内的[阿里云](https://www.aliyun.com/) 、[腾讯云](https://cloud.tencent.com/) ，国外的 [GoDaddy](https://sg.godaddy.com/) 都是不错的选择，这里以阿里云为例。

> 首页 --> 域名与网站 --> 域名注册

按自己喜好搜索购买域名就好：

![截屏2020-09-03 19.34.30](https://tva1.sinaimg.cn/large/007S8ZIlgy1gidoymuua8j31gt0u0e40.jpg)

购买完成后进入阿里云控制台点击 **域名** 进入域名控制台，在 **域名列表** 点击解析 **解析** 进入云解析 DNS：

![截屏2020-09-03 19.38.21](https://tva1.sinaimg.cn/large/007S8ZIlgy1gidp2lyr9jj31yg0qmgrt.jpg)

**_dnsauth** 是 SSL 证书相关配置，暂放一放，这里先如示例添加两组记录，分别是：

+ 记录类型：CNAME，主机记录：WWW，记录值：[username].github.io
+ 记录类型：CNAME，主机记录：@，记录值：[username].github.io

其他设置全部默认，回到 Github Pages 仓库的 **Settings**，把购买的域名填入自定义域名栏中：

![截屏2020-09-03 19.46.49](https://tva1.sinaimg.cn/large/007S8ZIlgy1gidpbe3jq7j31a70u0wko.jpg)

稍等十几分钟（长短视服务器负载决定）待服务器完成 DNS 自动配置，绑定域名部分就完成了。

# SSL 配置

这时打开博客网站，会发现浏览器显示（网址左边）显示 **连接不安全**，这是因为域名并没有配置 **SSL 证书**，选择很多，付费、免费的都有，这里就不一一罗列了，以阿里云为例。

进入阿里云控制台

> 产品与服务 --> SSL 证书 --> 概览 --> 购买证书

可以选择免费的 SSL 版本：

> 单个域名，DV 域名级 SSL，免费版

等待支付成功信息出现后，回到**证书控制台**可以看到证书列表中出现了待申请的 SSL 证书：

![截屏2020-09-03 20.00.28](https://tva1.sinaimg.cn/large/007S8ZIlgy1gidppmlm35j31vk0ewq5m.jpg)

点击 **证书申请**，填写已购买的域名及实名认证相关信息，提交后等待审核。

![截屏2020-09-03 20.02.23](https://tva1.sinaimg.cn/large/007S8ZIlgy1gidprpnkoaj31uo0bkq54.jpg)

可以看到已经审核完成的证书状态会显示 **已签发**。

这时，如果是使用阿里云自家的域名服务则回到域名的**云解析 DNS**设置栏会发现多出一项主机记录为**_dnsauth** 的记录，别家的域名则需要修改 DNS 服务器并手动添加记录，不多赘述。

![截屏2020-09-03 20.11.23](https://tva1.sinaimg.cn/large/007S8ZIlgy1gidq0zxejaj318u04475i.jpg)

好了，等一会后再访问博客网站网址左边那把喜闻乐见的小锁又回来了。

![截屏2020-09-03 20.18.42](https://tva1.sinaimg.cn/large/007S8ZIlgy1gidq8qgop2j30jg0giq6f.jpg)

# 备案

要想使用七牛云等图床的 https 链接，网站需要在国内备案以启用自定义 CDN 加速域名，否则域名加速范围只能覆盖海外：

![截屏2020-09-03 20.22.20](https://tva1.sinaimg.cn/large/007S8ZIlgy1gidqcc5xg9j31gx0u0n2p.jpg)

下面是转载至七牛云官方文档的网站公安备案和查询流程：

<https://developer.qiniu.com/fusion/kb/5992/the-web-site-public-security-registration-process>

# One More Thing

值得一提的是，如果在博客中插入以 http 明文传输的图片则会因为 **mixed content** 警告，网站同样会被浏览器判定为不安全，所以尽量选择支持 https 和多种图片格式的图床吧。

分享一个开源、免登录、游客限流但可升级付费 PRO 版本的图床：[imgurl](https://imgurl.org/)





