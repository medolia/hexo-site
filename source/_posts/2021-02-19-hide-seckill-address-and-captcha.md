---
title: 秒杀项目复盘（五）隐藏秒杀地址和图形验证码
date: 2021/2/19 21:01:50
categories: 编程
tags:
  - spring boot
  - redis
  - jquery
excerpt: 通过 redis 和 jquery ajax 实现秒杀项目中的秒杀地址隐藏和图形验证码
---

# 前言

项目链接：https://github.com/medolia/secondkill

# 实现细节

## 简述

1. 当秒杀倒计时结束后，商品处于可秒杀状态时，jquery 将验证码图片和答案输入框显示出来，通过图片的 `src` 属性生成图形验证码，redis 存入匹配的验证码答案。
2. 用户点击前端页面的 **秒杀** 按钮后，使用 jquery 的 ajax 发起一个 GET 请求，路径为 `/seckill/path`。
3. 检查答案是否正确（正确后需要删除对应的 redis 记录），之后生成 path uuid 存入 redis，返回成功信息。
4. 前端监测到已成功生成 path uuid 并以此为参发起 ajax POST 请求，路径为 `/seckill/{pathId}/do_seckill`。
5. 前端轮询，等待消息队列处理返回秒杀结果。

## 代码展示

### 秒杀按钮触发函数 getSeckillPath()

```javascript
    function getSeckillPath() {
        g_showLoading();
        $.ajax({
            url: "/seckill/path",
            type: "GET",
            data: {
                goodsId: $("#goodsId").val(),
                verifyCode: $("#verifyCode").val()
            },
            success: function (data) {
                if (data.code == 0) {
                    let path = data.data;
                    doSeckill(path);
                } else {
                    layer.msg(data.msg.msg);
                }
            },
            error: function () {
                layer.msg("客户端请求有误");
            }
        });
    }
```

### 秒杀请求函数 doSeckill()

```javascript
    function doSeckill(path) {
        $.ajax({
            url: "/seckill/" + path + "/do_seckill",
            type: "POST",
            data: {
                goodsId: $("#goodsId").val()
            },
            success: function (data) {
                if (data.code == 0) {
                    getSeckillResult($("#goodsId").val());
                } else {
                    layer.msg(data.msg.msg);
                }
            },
            error: function () {
                layer.msg("客户端请求出错")
            }
        })
    }
```

### 检测验证码答案，生成 path uuid

```java
    @RequestMapping(value = "/path", method = RequestMethod.GET)
    @ResponseBody
    // ...
    public Result<String> getSeckillPath(SeckillUser user, @RequestParam("goodsId") long goodsId,
                                         @RequestParam("verifyCode") int verifyCode) {
        if (user == null)
            return Result.error(CodeMsg.SESSION_ERROR);

        boolean check = seckillService.checkVerifyCode(user, goodsId, verifyCode);
        if (!check)
            return Result.error(CodeMsg.SECKILL_VERIFY_FAIL);

        String path = seckillService.createSeckillPath(user, goodsId);
        return Result.success(path);
    }
```

### 核心秒杀接口

```java
    @RequestMapping(value = "/{path}/do_seckill", method = RequestMethod.POST)
    @ResponseBody
    public Result<CodeMsg> seckill(SeckillUser user, @RequestParam("goodsId") long goodsId,
                                   @PathVariable("path") String path) {

        // 用户登录验证
        if (user == null) return Result.error(CodeMsg.SESSION_ERROR);

        // path 验证
        log.info("path: " + path);
        boolean check = seckillService.checkPath(user, goodsId, path);
        if (!check)
            return Result.error(CodeMsg.REQUEST_ILLEGAL);

        // 内存标记，减少 redis 访问
        boolean seckillOver = localOverMap.get(goodsId);
        log.info("whether seckill is over confirmed by local memory: " + seckillOver);
        if (seckillOver) {
            return Result.error(CodeMsg.SECKILL_OVER);
        }

        // 预减库存
        long stock = redisService.decr(GoodsKey.getSeckillGoodsStock, "" + goodsId);
        log.info("stock obtained from redis: " + stock);
        if (stock < 0) {
            localOverMap.put(goodsId, true);
            return Result.error(CodeMsg.SECKILL_OVER);
        }

        // 判断重复秒杀
        SeckillOrder order = orderService.getSeckillOrderByUserIdGoodsId(user.getId(), goodsId);
        log.info(order == null ? "new seckill order" : "seckill repeated");
        if (order != null)
            return Result.error(CodeMsg.SECKILL_REPEATED);

        // 入队
        SeckillMsg msg = new SeckillMsg();
        msg.setUser(user);
        msg.setGoodsId(goodsId);
        sender.sendSeckillMessage(msg);

        return Result.success(CodeMsg.SECKILL_WAIT);
    }
```