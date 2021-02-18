---
title: 秒杀项目复盘（二）redis key 设计
date: 2021/2/17 12:26:15
categories: 编程
tags:
  - 设计模式
  - redis
excerpt: 详解秒杀项目中 redis key 的设计细节
---

# 前言

项目链接：https://github.com/medolia/secondkill

# 实现细节

## 简述

1. 考虑所有键都应该存在过期时间和前缀，定义这样一个接口。
2. 创建一个抽象类包含通用的实现。
3. 所有子类继承自抽象类，并有各自特别的前缀。

即如图的继承关系
![截屏2021-02-17 14.16.49](https://tva1.sinaimg.cn/large/008eGmZEgy1gnqi9sfwmcj31ds0mgq49.jpg)

## 代码展示

### 接口 KeyPrefix

```java
public interface KeyPrefix {
    int expireSeconds();

    String getPrefix();
}
```

### 抽象类 BaseKeyPrefix

```java
public abstract class BaseKeyPrefix implements KeyPrefix {
    // ...

    public BaseKeyPrefix(String prefix) {
        this(0, prefix); // 0 代表永不过期
    }

    // ...

    public String getPrefix() {
        String className = this.getClass().getSimpleName();
        // 即键前缀为 <类名>:<自定义前缀>
        return className + ":" + prefix;
    }
}
```

### 子类例 SeckillUserKey

```java
public class SeckillUserKey extends BaseKeyPrefix {

    private static final int TOKEN_EXPIRE = 3600 * 24; // token 键过期时间为 1 天
    private static final int ID_EXPIRE = 3600 * 24 * 3; // id 键过期时间为 3 天

    // ... 
    
    // 即 token 键为 SeckillUserKey:tk<token值> 
    public static SeckillUserKey token = new SeckillUserKey(TOKEN_EXPIRE, "tk");
    public static SeckillUserKey getById = new SeckillUserKey(ID_EXPIRE, "id");
}
```

### RedisService set 方法

```java
public <T> boolean set(KeyPrefix prefix, String key, T value) {
        Jedis jedis = null;
        try {
            // ...

            // 拼接前缀生成真正的 key
            String realKey = prefix.getPrefix() + key;
            int seconds = prefix.expireSeconds();
            if (seconds <= 0) jedis.set(realKey, str);
            else jedis.setex(realKey, seconds, str);
            return true;
        } finally {
            returnToPool(jedis);
        }
    }
```
