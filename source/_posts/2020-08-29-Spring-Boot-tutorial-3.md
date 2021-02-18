---
title: Spring Boot 入门系列教程（三）
date: 2020/8/29 14:26:15
categories: 编程
tags: 
  - spring boot
  - spring security
  - spring JPA
  - thymeleaf
excerpt: 使用 Spring Security 配置用户认证服务
---

# 介绍

这部分教程将主要使用 Spring Data JPA 和 Spring Security 作为持久化用户数据、完成用户认证操作的方案。

# 用户领域对象及持久化

## 实现用户详情接口

来看新定义的 User 对象：

```java
@Data
@Entity
@NoArgsConstructor
@Table(name="user_info")
public class User implements UserDetails {

    @Id
    @GeneratedValue(strategy = GenerationType.AUTO)
    private int id;
    private String username;
    private String password;
    private Date create_date;
    private String authority;

    public User(String username, String password) {
        this.username = username;
        this.password = password;
        this.create_date = new Date();
        this.authority = "ROLE_USER";
    }

    @Override
    public Collection<? extends GrantedAuthority> getAuthorities() {
        return Arrays.asList(new SimpleGrantedAuthority("ROLE_USER"));
    }

    @Override
    public boolean isAccountNonExpired() {
        return true;
    }

    @Override
    public boolean isAccountNonLocked() {
        return true;
    }

    @Override
    public boolean isCredentialsNonExpired() {
        return true;
    }

    @Override
    public boolean isEnabled() {
        return true;
    }
}
```

可以看到，与之前不同的是，User 对象实现了Spring Security 定义的 **UserDetails** 接口，其中 **getAuthorities()** 方法表明用户表中的所有用户都被授予了“ROLE_USER”权限，其他方法如方法名定义用户是否过期、被锁定、认证过期、允许认证。

## 用户详情服务

接下来我们需要创建用户详情服务，其需要实现一个仅含一个方法的接口 **UserDetailsService**：

```java
@Service
public class UserRepoUserDetailServ implements UserDetailsService {

    private UserRepo userRepo;

    @Autowired
    public UserRepoUserDetailServ(UserRepo userRepo) {
        this.userRepo = userRepo;
    }

    @Override
    public UserDetails loadUserByUsername(String username) throws UsernameNotFoundException {
        User user = userRepo.findByUsername(username);
        if (user != null) {
            return user;
        }
        throw new UsernameNotFoundException("User '" + username + "' not found");
    }
}
```

简单粗暴，**@Service** 注解指定类为 Spring Bean，注入 User 对应的 repository，接口方法直接使用之前 Spring JPA 定义并自动实现的 findByUsername() 方法即可，然后若无此用户则抛出异常。 

# Spring Security 配置

## 主要配置

有了符合 Spring Security 的用户对象及对应的用户详情服务，接下来就是 Spring Security 的详细配置了。

首先是注入之前定义的用户详情服务：

```java
@Configuration
@EnableWebSecurity
public class SecurityConfig extends WebSecurityConfigurerAdapter {
    private UserDetailsService userDetailsService;

    @Autowired
    @Qualifier("userRepoUserDetailServ")
    public void setUserDetailsService(UserDetailsService userDetailsService) {
        this.userDetailsService = userDetailsService;
    }
```

注解 **@Configuration**、**@EnableWebSecurity** 及扩展类 **WebSecurityConfigurerAdapter** 使得 Spring 识别此类为 Spring Security 配置类，这里需要注意使用 **@Qualifier** 注解指定我们已经配制好的用户详情服务，否则 Spring 会因为目标注入的类里含有多个 Bean 而出错。

接着是配置一个密码转码器，这里使用的是 **Pbkdf2**，使数据库存储加密后的密码，而用户登录时 Spring 会自动使用解码器解码输入的密码与数据库核对。

```java
    @Bean // 标记为 Spring Bean
    public PasswordEncoder encoder() {
        return new Pbkdf2PasswordEncoder();
    }
```

好的，然后配置指定用户详情服务和密码转码器即可，通过实现 **configure(AuthenticationManagerBuilder auth)** 的签名方法：

```java
    @Override
    protected void configure(AuthenticationManagerBuilder auth) throws Exception {
        auth.userDetailsService(userDetailsService)
            .passwordEncoder(encoder());
    }
```

接着是配置对应用户权限允许访问的页面和登录、登出后的跳转页面，通过实现 **configure(HttpSecurity http)** 的签名方法：

```java
    @Override
    protected void configure(HttpSecurity http) throws Exception {
        http
            .authorizeRequests()
                .antMatchers("/vote", "/result").access("hasRole(\"ROLE_USER\")")
                .antMatchers("/", "/**").access("permitAll")

            .and()
                .formLogin()
                    .loginPage("/login").defaultSuccessUrl("/vote", true)

            .and()
                .logout().logoutSuccessUrl("/");
    }
```

可以看到 Spring Security 的 configure() 均适用**构造者（builder）**风格的接口来展现认证细节，首先是 **.authorizeRequests()** 指定用户权限对应的页面请求，这里需要注意顺序，越往后的配置优先级越低，比如这里配置的是拥有 “ROLE_USER” 权限的用户允许投票、查看投票结果，在接收到这两个页面的请求时如果用户未登录会自动跳转至登录页面；而除此之外的所有页面允许所有用户访问。

**and()** 表明两部分配置的过渡，**formLogin()** 部分指定登录链接和登录成功后的跳转至投票页面（其中参数 true 表明强制要求用户登录后定向至该页面），**logout()** 部分指定登出后跳转至主页。

## 自定义用户登录、注册

### 登录功能

默认状态 Spring Security 会提供一个简单的登录页面，如果需要自定义页面则需要在模板库中创建一个名为 **login** 的页面并声明为一个视图控制器。

比如其页面这样设计：

```html
<body>
<div class="container">

    <h1>Login Page</h1>

    <p>New here? Click <a th:href="@{/register}">here</a> to register</p>

    <form method="POST" th:action="@{/login}" id="loginForm">

        <label for="username">Username:</label>
        <input type="text" name="username" id="username">

        <label for="password">Password:</label>
        <input type="text" name="password" id="password">

        <input class="btn btn-primary" type="submit" value="Login">

    </form>

</div>
</body>
```

Spring 会自动识别页面中输入元素（input）名称（name）为 **username** 和 **password** 的元素，拦截 **“/login”** 表单提交请求。

接着声明控制器：

```java
@Configuration
public class WebConfig implements WebMvcConfigurer {

    @Override
    public void addViewControllers(ViewControllerRegistry registry) {
        registry.addViewController("/login");
    }
}
```

### 注册功能

Spring Security 并没有注册功能相关的支持，但实现起来并不难。

我们可以首先创建一个类来保存表单数据：

```java
@Data
public class RegisterForm {

    private String username;
    private String password;

    public User toUser(PasswordEncoder passwordEncoder) {
        return new User(username, passwordEncoder.encode(password));
    }
}
```

然后设计处理对应请求的控制器：

```java
@Controller
@RequestMapping("/register")
public class RegisterController {

    private UserRepo userRepo;
    private PasswordEncoder passwordEncoder;

    @Autowired
    public RegisterController(UserRepo userRepo, PasswordEncoder passwordEncoder) {
        this.userRepo = userRepo;
        this.passwordEncoder = passwordEncoder;
    }

    @GetMapping
    public String registerForm() {
        return "register";
    }

    @PostMapping
    public String processRegistration(RegisterForm form) {
        userRepo.save(form.toUser(passwordEncoder));
        return "redirect:/login";
    }

}
```

控制器定义 “/register” 的 “GET” 请求响应为跳转至 “register” 页面，其设计为：

```html
<body>

<div class="container">
    <div class="my-5 jumbotron border border-info">
        <h1>Create New Account</h1>
        <p>Quick and Simple</p>

        <form METHOD="POST" th:action="@{/register}">

            <label for="username">Username:</label>
            <input type="text" name="username" id="username">

            <label for="password">Password:</label>
            <input type="text" name="password" id="password">

            <input class="btn btn-primary" type="submit" value="注册">

        </form>
    </div>
</div>

</body>
```

而 “/register” 的 “POST” 请求响应，这里注意是通过引入一个创建的 **POJO** 来获取前端页面参数，前段 input 的 **name** 与对象的字段名一一对应并自动绑定，用户注册表单提交为数据库记录数据操作，密码使用之前定义的转码器，之后重定向至登录页面。

### 用户登出

退出功能非常简单，只需要在需要有退出选择的页面增加一个 **“/logout”** 请求的表单即可:

```html
<form method="POST" th:action="@{/logout}">
  <input type="submit" value="退出登录">
</form>
```

Spring Security 会拦截对 **“/logout”** 的请求，用户点击按钮时会清理他们的 session，并重定向至之前配置定义的主页面。

# 总结

到这里，这系列教程就结束了，我们得到了一个完整的基于 Spring Boot 的小型问卷调查系统，它有基本的投票、展示投票结果的功能，也支持用户认证、注册和登录。希望对你熟悉 Spring Boot 有所帮助。

当然，这样的系统只是一个雏形，Spring Boot 的广袤自定义配置仅仅只展示了冰山一角，系统还可以新增很多功能，比如展示投票结果时可以过滤掉重复 ip 的无效票，在投票页面设计一个倒计时等等，可能有时间的话我会弄弄并把对应教程发到博客上吧。

最后是该系列教程对应的 [Github 源码](https://github.com/medolia/QueVoteSys)。
