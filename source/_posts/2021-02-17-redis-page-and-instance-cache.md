---
title: 秒杀项目复盘（四）对象与页面缓存
date: 2021/2/17 16:26:15
categories: 编程
tags:
  - redis
  - spring boot
excerpt: 复盘秒杀项目中的对象与页面缓存
---

## 前言

项目链接：https://github.com/medolia/secondkill

在介绍项目中对象和页面缓存之前，有必要介绍一下常用的缓存更新策略，参考一篇介绍缓存更新的[高质量博客文章](https://blog.csdn.net/tTU1EvLDeLFq5btqiK/article/details/78693323)

这里我们选择最常用也比较实用的 **Cache Aside Pattern**
+ **失效**：应用程序先从cache取数据，没有得到，则从数据库中取数据，成功后，放到缓存中。
+ **命中**：应用程序从cache中取数据，取到后返回。
+ **更新**：先把数据存到数据库中，成功后，再让缓存失效。

![](https://medoliablog.oss-cn-hangzhou.aliyuncs.com/2021/02/17/16135560493097.jpg)

## 实现细节

### 对象缓存
+ id 用户对象缓存，在登录时判断手机号是否存在、使用 id 查询时触发

```java
public SeckillUser getById(long id) {
        // 查询对象缓存
        SeckillUser user = redisService.get(SeckillUserKey.getById, ""+id, SeckillUser.class);
        if (user != null) {
            return user;
        }

        // 若缓存未命中，查询数据库、更新缓存
        user = seckillUserDao.getById(id);
        if (user != null)
            redisService.set(SeckillUserKey.getById, ""+id, user);

        return user;
    }
```

+ token 用户对象缓存，在登录时登录成功后、添加 cookie 时触发

```java
private void addCookie(HttpServletResponse response, String token, SeckillUser user) {
        redisService.set(SeckillUserKey.token, token, user);
        Cookie cookie = new Cookie(COOKIE_NAME_TOKEN, token);
        cookie.setPath("/");
        cookie.setMaxAge(SeckillUserKey.token.expireSeconds());
        response.addCookie(cookie);
    }
```

### 页面缓存

缓存常访问的秒杀商品列表，注意这样的缓存一般过期时间很短，本项目设置为 60 秒

```java
    @RequestMapping(value = "/to_list", produces = "text/html")
    @ResponseBody
    public String list(HttpServletRequest request, HttpServletResponse response, Model model) { 
        // 取缓存
        String html = redisService.get(GoodsKey.getGoodsList, "", String.class);
        if (!StringUtils.isEmpty(html)) return html;

        // 手动渲染
        List<GoodsVo> goodsList = goodsService.listGoodsVo();
        model.addAttribute("goodsList", goodsList);
        WebContext ctx = new WebContext(request, response, request.getServletContext(),
                request.getLocale(), model.asMap());
        html = thymeleafViewResolver.getTemplateEngine().process("goods_list", ctx);
        if (!StringUtils.isEmpty(html)) // 加入缓存
            redisService.set(GoodsKey.getGoodsList, "", html);
        return html;
    }
```







