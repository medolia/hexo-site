---
title: Spring Boot 入门系列教程（一）
date: 2020/8/27 14:26:15
categories: 编程
tags: 
  - spring
  - spring JPA
  - MySQL
excerpt: 初步确定一个调查问卷系统的数据库结构，创建对应的领域对象和数据库交互操作对象

---

# 前言 <span id="1">

## 介绍 <span id="1.1">

试着做一个基于 Spring 的简单问卷调查系统！这是一个旨在帮助刚入门 Spring 框架的新手熟悉 Spring Boot 的运作方式，初步理解 Spring MVC 的工作原理的系列教程，教程有点长，但我不会鸽。

## 环境要求 <span id="1.2">

+ IDE: Intellij IDEA
+ JDK: oracle-jdk, Java 8
+ 数据库: MYSQL 5.7
+ 项目管理: Maven 3

## 使用依赖 <span id="1.3">

+ thymeleaf: 模板解析引擎
+ Spring Web: 基础的Spring MVC功能
+ Spring Security: 提供用户注册、登录、页面权限认证管理等服务
+ Spring JPA: 持久化方案
+ Devtools: 配合chrome的live load插件实现热部署
+ Mysql: 实现mysql数据库连接、读取、插入数据等操作
+ lombok: 提供一些实用的注解

## 创建项目 <span id="1.4">

> Intellij IDEA --> new --> project--> Spring Initializr 

![截屏2020-08-28 09.31.12](https://tva1.sinaimg.cn/large/007S8ZIlgy1gi7sssf7qjj31hv0u0af3.jpg) 

项目名、描述、包名根据自己喜好定义就好啦。 

> --> Next

![截屏2020-08-28 09.34.59](https://tva1.sinaimg.cn/large/007S8ZIlgy1gi7sst8og6j31gj0u0qbg.jpg)

选择这些项目所需的依赖，推荐一开始先不注入 Spring Security，待需要实现用户认证时再增加这个依赖。

> --> Next --> Finish

## 项目结构 <span id="1.5">

+ Java 

  + {package name}

    + data - 数据库操作相关对象
    + model - 领域实体对象
    + security - 用户登录认证等配置对象
    + web - 各种控制网页跳转响应的控制器

  + resources

    + static

      + css - 样式表
      + fonts -字体
      + img - 图片
      + js - JavaScript 文件

    + templates - 模板页面

      

# 构建数据库 <span id="2">

## 数据库结构 <span id="2.1">

既然目标是做一个问卷调查系统，我习惯第一步把数据库结构考虑好。

我创建了四个表，分别是：

**问题表**

``` sql
CREATE TABLE `question_info` 
(
    `id`                   int(50)      NOT NULL, # 主键
    `question_name`        varchar(255) NOT NULL, # 问题
    `question_description` varchar(255) DEFAULT NULL # 问题描述
) ENGINE = InnoDB
  DEFAULT CHARSET = utf8;
```

**选项表**

```sql
CREATE TABLE `option_info`
(
    `id`             int(50)      NOT NULL,
    `question_id`    int(50)      NOT NULL,
    `option_content` varchar(255) NOT NULL
) ENGINE = InnoDB
  DEFAULT CHARSET = utf8;
```

**答案表**

```sql
CREATE TABLE `answer_info`
(
    `id`             int(50)      NOT NULL,
    `question_id`    int(50)      NOT NULL,
    `option_id`      int(50)      NOT NULL,
    `user_id`        int(50)      NOT NULL DEFAULT '0',
    `answer_content` varchar(255)          DEFAULT NULL,
    `create_ip`      varchar(255) NOT NULL,
    `create_date`    timestamp    NOT NULL DEFAULT CURRENT_TIMESTAMP
) ENGINE = InnoDB
  DEFAULT CHARSET = utf8;
```

**用户表**

```sql
CREATE TABLE `user_info`
(
    `id`          int(50)      NOT NULL,
    `username`    varchar(255) NOT NULL,
    `password`    varchar(255) NOT NULL,
    `create_date` timestamp    NOT NULL DEFAULT CURRENT_TIMESTAMP,
    `authority` 	varchar(255) DEFAULT NULL
) ENGINE = InnoDB
  DEFAULT CHARSET = utf8;
```

各表关系如图:

![截屏2020-08-27 16.35.08](https://tva1.sinaimg.cn/large/007S8ZIlgy1gi7ssrery2j31f00o278g.jpg)

## 配置连接数据库 <span id="2.2">

构建完数据库后需要配置数据库连接项目和数据库了，一般必要的是数据库链接、数据库用户名与密码。

```properties
spring.datasource.url=jdbc:mysql://localhost:8889/quevote_db
spring.datasource.username=root
spring.datasource.password=root
#是否在控制台打印JPA执行过程生成的SQL
spring.jpa.show-sql=true
#表示JPA对应的数据库是MySQL
spring.jpa.database=mysql
#表示在项目启动时根据实体类更新数据库中的表
spring.jpa.hibernate.ddl-auto=update
#表示使用的数据库方言是MySQL57Dialect
spring.jpa.properties.hibernate.dialect=org.hibernate.dialect.MySQL57Dialect
```

Spring Boot 会自动识别驱动类型，一般不用显式指定 driver 类型。

# 使用 Spring JPA 持久化数据 <span id="3">

## 实体化领域对象 <span id="3.1">

这里需要为每一个表创建一个领域（Domain）对象，并指定为实体被 Spring JPA 识别，数据库与实体对象字段的类型和名称必须严格一一对应。

```java
@Data
@Entity
@NoArgsConstructor
@Table(name="question_info")
public class Question {

    @Id
    @GeneratedValue(strategy = GenerationType.AUTO)
    private int id;
    private String question_name;
    private String question_description;

}
```

这里依次介绍一下使用的4个类注解与2个字段注解，**@Data** 为对象自动生成所有的 getter 和 setter 方法，还会为所有 final 字段生成构造器，**@Entity** 如名指定对象为实体被 Spring JPA 识别，**@NoArgsConstructor** 生成一个无字段的构造器（Spring JPA 要求所有实体必须要有一个无字段构造器），**@Table** 指定对应表名，**@Id** 和**@GeneratedValue** 指定实体 id 字段并表明 id 值自动递增。

其他三个对象大同小异，修改一下对应表的名字和字段定义就行了，这里不多赘述。

```java
@Data
@NoArgsConstructor
@Entity
@Table(name="answer_info")
public class Answer {

    @Id
    @GeneratedValue(strategy = GenerationType.AUTO)
    private int id;
    private int question_id;
    private int option_id;
    private int user_id;
    private String answer_content;
    private String create_ip;
    private Date create_date;
    
    @PrePersist // 预先定义日期为当前时间 
    void createdAt() {
        this.create_date = new Date();
    }
		
    // 自定义一个额外的构造器
    public Answer(int question_id, int option_id, String create_ip) {
        this.question_id = question_id;
        this.option_id = option_id;
        this.create_ip = create_ip;
    }

}
```

```java
@Data
@NoArgsConstructor
@Entity
@Table(name="option_info")
public class Option {

    @Id
    @GeneratedValue(strategy = GenerationType.AUTO)
    private int id;
    private int question_id;
    private String option_content;

}
```

```java
@Data
@Entity
@NoArgsConstructor
@Table(name="user_info")
public class User {

    @Id
    @GeneratedValue(strategy = GenerationType.AUTO)
    private int id;
    private String username;
    private String password;
    private Date create_date;
    private String authority;
		
    // 额外的构造器
    public User(String username, String password) {
        this.username = username;
        this.password = password;
        this.create_date = new Date();
        this.authority = "ROLE_USER";
    }

}
```



## 声明 JPA repository <span id="3.3">

如果说领域对象实体对应数据库的每一行的话，那么即将声明的 Spring JPA repositoty 就是对应数据库中的每一个表，Spring JPA 的显式声明十分简单，只需要创建一个接口类然后扩展一个由 Spring JPA 定义的接口即可。

```java
import com.que.votesys.model.User;
import org.springframework.data.repository.CrudRepository;

public interface UserRepo extends CrudRepository<User, Integer> {

    User findByUsername(String username);

}
```

**CrudRepository** 提供了基本的CRUD（创建，读取，更新，删除）操作的方法，第一个参数指定持久化的领域实体，第二个参数是实体ID属性类型。神奇的是，Spring Data JPA 完全不用编写实现类，所有的十多个方法都是开箱即食！此外，我们还可以额外编写自定义 JPA 方法，比如这里定义的 **findByUsername** 方法，Spring Data 会自动解析是要获取（find）一个（返回类型为 User）以（By）用户名（Username）为特征的实体。

其他 repository 声明类似，这里不多说。

```java
import com.que.votesys.model.Answer;
import org.springframework.data.repository.CrudRepository;

public interface AnswerRepo extends CrudRepository<Answer, Integer> {

}
```

```java
import com.que.votesys.model.Option;
import org.springframework.data.repository.CrudRepository;

public interface OptionRepo extends CrudRepository<Option, Integer> {

}
```

```java
import com.que.votesys.model.Question;
import org.springframework.data.repository.CrudRepository;

public interface QuestionRepo extends CrudRepository<Question, Integer> {
    
}
```

# 小结 <span id="4">

好的现在我们基本搞定了数据库的持久方案，目前的项目结构如下: 

![截屏2020-08-28 09.41.24](https://tva1.sinaimg.cn/large/007S8ZIlgy1gi7ssqglhgj30n20de0tk.jpg)

你可能会觉得这个项目一开始折腾这么久连运行都不行有点不解。别急，对于一个问卷调查系统，数据库模块毫无疑问至关重要，我们在之后的教程可以看到一个好的数据库结构在模版解析，数据交互等过程中是多么重要。
