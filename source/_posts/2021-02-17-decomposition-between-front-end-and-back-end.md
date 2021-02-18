---
title: 秒杀项目复盘（三）前后端分离
date: 2021/2/17 14:26:15
categories: 编程
tags:
  - jquery
  - spring boot
  - spring MVC
excerpt: 详解秒杀项目中的前后端分离
---

# 前言

项目链接：https://github.com/medolia/secondkill

Spring MVC 已经将后端项目分为 Service（业务层）、Dao（数据库操作）、Entity（实体）、Controller（控制层）等，而一般 Controller 负责将数据注入到 Model 里再返回到前端，由页面渲染引擎解析 Model 生成渲染页面。

但这样流程的前后端耦合度仍然不够低，我们希望后端返回的是一个 json、xml 等格式的轻量级资源，接着在前端借助 jquery、angularJS、vue 等框架消费得到的资源，使前后端进一步解耦。

# 实现细节

## 简述

1. 定义一个 CodeMsg 类携带当处理出错时的错误信息，包括 int 类型的错误码和 String 类型的错误信息文本。
2. 定义一个 Result 类作为携带前后端交互信息的主要载体，包含 int 类型的响应码，CodeMsg 类的错误信息，泛型类的数据。规定当请求成功时，响应码为 0，错误信息为 null，数据不为 null；而请求失败时，响应码为错误码，错误信息不为 null，数据为 null。

## 代码展示

### 错误信息载体类 CodeMsg

```java
public class CodeMsg {

    private int code;
    private String msg;
    
    // ...

    //通用的错误码
    public static CodeMsg SUCCESS = new CodeMsg(0, "success");
    public static CodeMsg SERVER_ERROR = new CodeMsg(500100, "服务端异常");
    // ...

    //登录模块 5002XX
    public static CodeMsg SESSION_ERROR = new CodeMsg(500210, "Session不存在或者已经失效");
    public static CodeMsg PASSWORD_EMPTY = new CodeMsg(500211, "登录密码不能为空");
    // ...

    //商品模块 5003XX
    //订单模块 5004XX
    //秒杀模块 5005XX
    // ...

    @Override
    public String toString() {
        return "CodeMsg [code=" + code + ", msg=" + msg + "]";
    }
```

可以看到每个模块的错误码前缀会不一样，发生错误时可根据错误码定位是哪个环节发生故障。

### 前后端信息交互载体类 Result

```java
public class Result<T> {
    private static final int SUCCESS_CODE = 0;

    private int code;
    private CodeMsg msg;
    private T data;

    public Result(int code, CodeMsg msg) {
        this.code = code;
        this.msg = msg;
    }

    public Result(int code, T data) {
        this.code = code;
        this.data = data;
    }

    // 成功获得结果时
    public static <T> Result<T> success(T data) {
        return new Result<T>(SUCCESS_CODE, data);
    }

    // 出现错误时
    public static <T> Result<T> error(CodeMsg msg) {
        return new Result<T>(msg.getCode(), msg);
    }
}
```

### Controller 类示例

```java
@Controller
@RequestMapping("/demo")
public class SampleController {

    // ...

    @RequestMapping("/mq")
    @ResponseBody // 此注解使后端返回一个可消费的资源而不是默认的逻辑视图名
    public Result<String> mq() {
        log.info("msg sent: test msg.");
        sender.sendTestMsg("test msg");
        return Result.success("msg sent!");
    }
```

### 前端 jquery 消费资源例

```javascript
function doLogin(){
	g_showLoading();

	// ...
	
	$.ajax({
		url: "/login/do_login",
	    type: "POST",
	    data:{
	    	mobile:$("#mobile").val(),
	    	password: password
	    },
	    success:function(data){
	    	layer.closeAll();
	    	if(data.code == 0){
	    		layer.msg("成功");
	    		window.location.href="/goods/to_list";
	    	}else{ // 响应码不为 0，说明请求失败，弹出含有错误信息文本的窗口
	    		layer.msg(data.msg.msg);
	    	}
	    },
	    error:function(){
	    	layer.closeAll();
	    }
	});
}
```
