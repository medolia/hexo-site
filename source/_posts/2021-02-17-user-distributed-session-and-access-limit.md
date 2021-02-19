---
title: 秒杀项目复盘（一）用户分布式 session 与接口限流
date: 2021/2/17 10:26:15
categories: 编程
tags:
  - 分布式
  - spring boot
  - redis
excerpt: 通过 redis 实现秒杀项目中的分布式 session 和接口限流
---

# 前言

项目链接：https://github.com/medolia/secondkill

# 实现细节

## 简述

### 分布式 session

1. 用户在登录时，会生成一个 **token** 存入 redis 中，有效时间定为 1 天。
2. 实现一个自定义注解和配套的注解拦截器，在方法执行前会将 User 对象注入一个**线程局部变量**内。
3. 自定义一个**变量解析器**，当发现 controller 的入参中有 User 类时会将局部变量内的对象与该入参绑定。

### 接口限流

1. 同样是注解拦截器中的方法执行前操作，获取自定义注解中的定义的时间和点击次数限制量。
2. 若第一次访问，则将包含请求 URI 的键和已点击次数的值存入 redis，过期时间为注解中获得的时间。
3. 若已访问过，尝试自增一次键值，若已大于限制量则返回访问次数达上限的错误信息。

## 代码展示

### UserService 类的 login 方法

```java
    public String login(HttpServletResponse response, @Valid LoginVo loginVo) {

        // 判断手机号是否存在
        // ...

        // 核对密码
        // ...

        // 生成 cookie，即使用一个统一的标识键、一个 uuid 为值存入 cookie 中
        String token = UUIDUtil.uuid();
        addCookie(response, token, user);
        return token;
    }
    
    private void addCookie(HttpServletResponse response, String token, SeckillUser user) {
        // token 为键 user 为值存入 redis
        redisService.set(SeckillUserKey.token, token, user);
        Cookie cookie = new Cookie(COOKIE_NAME_TOKEN, token);
        cookie.setPath("/");
        cookie.setMaxAge(SeckillUserKey.token.expireSeconds());
        response.addCookie(cookie);
    }
    
```

### 自定义注解和配套的注解拦截器

自定义注解 AccessLimit

```java
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.METHOD)
public @interface AccessLimit {
    // 即限制在 seconds 秒内最多访问 maxCount 次
    int seconds();
    int maxCount();
    // 是否需要登录，默认为需要
    boolean needLogin() default true;
}
```

配套的注解拦截器 AccessInterceptor （已隐藏部分代码）

```java
public class AccessInterceptor implements HandlerInterceptor {

    @Override // 方法执行前解析注解
    public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler) throws Exception {
        if (handler instanceof HandlerMethod) { // 若为控制器方法
            // 从 cookie 中获取 token，再根据 token 从 redis 中获取 user
            SeckillUser user = getUser(request, response);
            // 将获取到的 user 实例绑定到线程局部变量中
            UserContext.setUser(user);

            HandlerMethod hm = (HandlerMethod) handler;

            // 获取所有注解值
            AccessLimit accessLimit = hm.getMethodAnnotation(AccessLimit.class);
            if (accessLimit == null)
                return true;
            int seconds = accessLimit.seconds();
            int maxCount = accessLimit.maxCount();
            boolean needLogin = accessLimit.needLogin();
            String key = request.getRequestURI();
            if (needLogin) { // 如果需要登录而用户不存在，返回需要登录的错误信息
                if (user == null) {
                    render(response, CodeMsg.SESSION_ERROR);
                    return false;
                }
                key += "_" + user.getId();
            }

            // 使用缓存进行节流，从第一次访问时开始计时，过期前访问次数达到上限返回 访问过于频繁的错误信息
            AccessKey ak = AccessKey.withExpire(seconds);
            Integer count = redisService.get(ak, key, Integer.class);
            if (count == null)
                redisService.set(ak, key, 1);
            else if (count < maxCount)
                redisService.incr(ak, key);
            else {
                render(response, CodeMsg.ACCESS_LIMIT_REACHED);
                return false;
            }

        return true;
    }

```

### 自定义变量解析器

```java
public class UserArgumentResolver implements HandlerMethodArgumentResolver {

    // ...

    @Override // 当传入参存在类型为 SeckillUser 的变量时触发自定义解析
    public boolean supportsParameter(MethodParameter methodParameter) {
        Class<?> clazz = methodParameter.getParameterType();
        return clazz==SeckillUser.class;
    }

    @Override
    public Object resolveArgument(MethodParameter methodParameter, ModelAndViewContainer modelAndViewContainer,
          NativeWebRequest nativeWebRequest, WebDataBinderFactory webDataBinderFactory) throws Exception {
        // 返回线程局部变量中的 user
        return UserContext.getUser();
    }

}
```

### 秒杀接口加入注解


```java
    @RequestMapping(value = "/path", method = RequestMethod.GET)
    @ResponseBody
    @AccessLimit(seconds = 5, maxCount = 5, needLogin = true)
    public Result<String> getSeckillPath(//...) {
        // ...
    }
```



