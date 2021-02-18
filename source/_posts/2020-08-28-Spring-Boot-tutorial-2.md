---
title: Spring Boot 入门系列教程（二）
date: 2020/8/28 14:26:15
categories: 编程
tags: 
  - spring
  - spring boot
  - spring JPA
  - spring MVC
  - thymeleaf
  - jQuery
excerpt: 实现展现投票页面、处理提交的投票表单和展示投票结果功能
---

# 前言

到这里我们就要开始构建展现层了，这部分可以展示 Spring MVC 是怎么调用模型，传入视图以及由控制器控制前端交互的。

## 准备工作

在此之前，我们需要做一些准备工作。

首先是确定问卷主题并插入数据，这里假设是评价某个游戏，从四个方面评价，每个游戏特性分别对应若干单选项。

```sql
--
-- 转存表中的数据 `question_info`
--

INSERT INTO `question_info` (`id`, `question_name`, `question_description`)
VALUES (0, 'Story Narrative', '叙事手法'),
       (1, 'Freedom of Game', '自由度'),
       (2, 'Philosophy of Art', '艺术哲学深度'),
       (3, 'Picture Quality', '画质');
       
--
-- 转存表中的数据 `option_info`
--

INSERT INTO `option_info` (`id`, `question_id`, `option_content`)
VALUES (0, 0, 'Touching'),
       (1, 0, 'Ordinary'),
       (2, 0, 'Just so so'),
       (3, 1, 'High'),
       (4, 1, 'Medium'),
       (5, 1, 'Low'),
       (6, 2, 'Unique'),
       (7, 2, 'Common'),
       (8, 2, 'Low level'),
       (9, 3, 'Close to Reality'),
       (10, 3, 'Enough for a Game'),
       (11, 3, 'Disappointing');       
```

然后是准备好 Bootstrap 和 jQuery 的文件用于美化页面和前端交互，当然你也可以直接指定链接（如接下来我所展示的）来引用。

![截屏2020-08-28 13.31.17](https://tva1.sinaimg.cn/large/007S8ZIlgy1gi7stjq7kmj30lg0amwf3.jpg)

准备工作结束。

# 控制器与页面设计

由于目标是小型的问卷调查系统，这里我们选择把所有响应链接放置在一个类中，先实现 **投票-->提交-->展示结果** 功能

```java
@Controller
@RequestMapping
public class MainController {

    private OptionRepo optionRepo;
    private QuestionRepo questionRepo;
    private AnswerRepo answerRepo;

    @Autowired
    public MainController(OptionRepo optionRepo, QuestionRepo questionRepo, AnswerRepo answerRepo) {
        this.optionRepo = optionRepo;
        this.questionRepo = questionRepo;
        this.answerRepo = answerRepo;
    }
}
```

**@Controller**与 **@RequestMapping** 两个类注解告诉 Spring Boot 这个类是一个会识别各种链接的控制器。首先注入三个主要 repository 用于数据库交互，**@Autowired** 可以注解一个方法或者构造器，表明 Spring app 运行时会识别此注解并将被注解的方法或构造器加入到 Spring Bean 当中。

## 主页设计

首先做一个运行 app 后第一个看到的页面，即主页。

```java
    @GetMapping("/")
    public String showIndex() {
        return "index";
    }
```

这里表明该方法会捕获 “/” 链接并返回模版库中一个名为“index”的模板，那么在项目中的 templates 文件夹中，设计名为“index”的如下主页：

```html
<!DOCTYPE html>
<html xmlns="http://www.w3.org/1999/xhtml"
      xmlns:th="http://www.thymeleaf.org">
<head>
    <title>Index Page</title>
    <meta charset="utf-8">
    <meta name="viewport" content="width=device-width, initial-scale=1">
    <link rel="stylesheet" href="https://maxcdn.bootstrapcdn.com/bootstrap/4.5.2/css/bootstrap.min.css">
    <script src="https://ajax.googleapis.com/ajax/libs/jquery/3.5.1/jquery.min.js"></script>
    <script src="https://cdnjs.cloudflare.com/ajax/libs/popper.js/1.16.0/umd/popper.min.js"></script>
    <script src="https://maxcdn.bootstrapcdn.com/bootstrap/4.5.2/js/bootstrap.min.js"></script>
</head>
<body>

<div class="container my-5 text-center">
    <div class="jumbotron border border-success">
        <h1>Vote for the Game: <strong>Sekiro</strong></h1>
        <p class="text-secondary">show your view relating to this game about</p>
        <ul class="list-group list-group-horizontal justify-content-center">
            <li class="list-group-item">FREEDOM</li>
            <li class="list-group-item">PICTURE QUALITY</li>
            <li class="list-group-item">STORY NARRATIVE</li>
            <li class="list-group-item">PHILOSOPHY OF ART</li>
        </ul>
        <a th:href="@{/vote}" class="mt-5 btn btn-success" role="button">Customize My Options</a>
    </div>
</div>

</body>
</html>
```

首先是开头，我们需要指定 thymeleaf 命名空间和引用 jQuery 和 Bootstrap 等css、js 文件，这里使用的是 mdn 链接，当然也可以指定项目文件夹中的文件，比如 bootstrap 的 css 文件就可以这样表示：

> ```html
> <link rel="stylesheet" th:href="@{/css/bootstrap.min.css}">
> ```

当然一定要注意引用的顺序，一般是：

> Bootstrap.css --> jQuery.js --> popper.js --> Bootstrap.js

那么我们就做成了这样的主页：

![截屏2020-08-28 13.57.01](https://tva1.sinaimg.cn/large/007S8ZIlgy1gi7stihcu7j31zm0pa779.jpg)

## 投票页面设计

主页的 html 表明点击 **Customize My Options** 按钮会跳转到 “/vote” 链接，回到 MainController 控制器，对应响应方法如下：

```java
    // 投票主页响应，返回一个答案-选项集的映射
    @GetMapping("/vote")
    public String showVoteForm(Model model) {
        List<Question> questionList = new ArrayList<>();
        questionRepo.findAll().forEach(i -> questionList.add(i));

        Map<Question, List<Option>> optionsMap = new HashMap<>();
        List<Option> optionList = new ArrayList<>();
        optionRepo.findAll().forEach(i -> optionList.add(i));
        for (int i = 0; i < questionList.size(); i++) {
            optionsMap.put(questionList.get(i), filterByQuestionId(optionList, questionList.get(i).getId()));
        }
        model.addAttribute("optionsMap", optionsMap);

        return "vote";
    }

    private List<Option> filterByQuestionId(List<Option> options, int Qid) {
        return options.stream().filter(x -> x.getQuestion_id() == Qid).collect(Collectors.toList());
    }
```

注意到方法中有个类型为 Model 的模型变量 model，在 Spring MVC 中它负责存储将要传入视图的各种信息。这里模型存储的是一个答案-选项列表的映射，具体步骤为先使用 JPA 接口定义的 **findAll()** 方法读取所有问题和选项然后存入两个列表中，然后将每个问题及过滤方法得出的对应选项集存入映射中，model 的 **addAttribute()** 方法指明变量名及对应的数据，最后返回 “vote” 视图，其页面设计如下：

```html
<body>

<div class="container my-5">
    <div class="jumbotron bg-light border border-primary">

        <h1>Demonstration How You Like the Game: <strong>Sekiro</strong></h1>
        <div class="progress">
            <div class="progress-bar bg-primary progress-bar-striped progress-bar-animated" id="vote-timer" style="width: 100%;"></div>
        </div>

        <form th:action="@{/votepost}" method="POST">

            <div class="form-group my-5" th:each="optionsForAQ : ${optionsMap}">
                <h4 th:text="${optionsForAQ.key.question_name}">QUESTION NAME HERE</h4>
                <div class="form-check" th:each="option : ${optionsForAQ.value}">
                    <input type="radio" class="form-check-input" th:id="'question' + ${optionsForAQ.key.id} + 'option' + ${option.id}" th:name="${optionsForAQ.key.id}+ '#' +${optionsForAQ.key.question_name}" th:value="${option.id}+ '#' +${option.option_content}">
                    <label class="form-check-label" th:for="'question' + ${optionsForAQ.key.id} + 'option' + ${option.id}" th:text="${option.option_content}"></label>
                </div>
            </div>

            <button class="btn btn-primary" id="submit-btn" type="submit">提交</button>
            <button class="btn btn-secondary" type="reset" >重置</button>

        </form>

    </div>
</div>

</body>
```

开头部分与 index 相同，不作多余展示，这里要着重说明 thymeleaf 的模板渲染原理。

来看表单部分，**th:text** 定义替代目标元素文本的值，div 中的 **th:each** 表示对于 optionsMap 中的每一组键值对均会生成一个 div 元素，thymeleaf 遍历映射时可以很方便的通过分别访问键值对对象的 **key** 和 **value** 属性得到键与值，于是我们对每个键的值（也就是 List\<Option>）再使用一次 **th:each** 迭代显示出选项，选项设置为单选，id 与 name 设置为问题编号+选项编号，value 设置为选项编号+选项内容。

thymeleaf 会如我们所愿渲染出这样的页面：

![截屏2020-08-28 14.38.49](https://tva1.sinaimg.cn/large/007S8ZIlgy1gi7stewhpfj313b0u0n1o.jpg)

## 投票表单处理

投票页面表明表单将有捕获 “/votepost” 的控制器处理，对应方法如下：

```java
    // 投票判定响应
    @PostMapping("/votepost")
    public String processVote(HttpServletRequest request) {
        List<Answer> answerList = new ArrayList<>();
        Enumeration<String> enu = request.getParameterNames();

        while (enu.hasMoreElements()) {
            String QAStr = enu.nextElement();
            String pattern = "\\d#\\w*";
            boolean isValidQAStr = Pattern.matches(pattern, QAStr);
            if (isValidQAStr) {
                int Qid = Integer.parseInt(QAStr.split("#")[0]);
                int OptId = Integer.parseInt(request.getParameter(QAStr).split("#")[0]);
                answerList.add(new Answer(Qid, OptId, request.getLocalAddr()));
            }

        }
        answerRepo.saveAll(answerList);

        return "votesuccess";
    }
```

表单指定 action 为 “POST”，所以对应捕获注解要改为 **@PostMapping** ，这里传入一个存储了表单信息的 HttpServletRequest 变量，具体信息可以提交表单后用浏览器自带的开发者工具看到。

比如我用的Chrome，导航: 

> 开发者工具 --> Network --> votepost --> Headers --> Form Data 

可以看到

![截屏2020-08-28 14.52.04](https://tva1.sinaimg.cn/large/007S8ZIlgy1gi7stfn202j30he08k0to.jpg)

_csrf 是 Spring Security 为防范 CSRF 攻击使用的 token ，而其他变量就是已选中的 input 中的 name 和 value 属性，这些数据将存入 request 中。使用 **getParameterNames()** 可以得到一个枚举变量，**getParameter()** 可以得到对应变量的值，这里使用了正则表达式过滤出有效的答案变量，然后 JPA 接口定义的 **saveAll()** 方法可以接受一个列表一次性存入列表中的所有对象，此外，request 的 **getLocalAddr()** 方法可以得到用户当前的 ip 地址。

## 展示投票结果

暂时不考虑选项不全的无效表单，设计返回的 votesuccess 页面：

```html
<body>

<div class="container my-5">
    <div class="jumbotron border border-success">
        <div class="row">
            <div class="col-4 d-flex justify-content-end">
                <span style="color: #249D3C">
                    <i class="far fa-check-square fa-7x"></i>
                </span>
            </div>
            <div class="col-8">
                <h1 class="text-success">Vote Success!</h1>
                <p class="mb-3 text-secondary">your vote is valid and has been recorded</p>
                <a class="btn btn-primary mr-3" th:href="@{/result}">Current Voting Result</a>           
            </div>
        </div>

    </div>
</div>

</body>
```

“/result” 对应的控制器方法有些特别，我们需要复用 optionsMap，而模型对象 model 的寿命只有一个页面，所以需要将其存入 session 中，Spring 提供了这样的类注解：

> ```java
> @SessionAttributes("optionsMap")
> ```

然后在方法中使用变量注解 **@ModelAttribute** 即可：

```java
    // 问卷结果页面
    @GetMapping("/result")
    public String showVoteDiagram(@ModelAttribute("optionsMap") Map<Question, List<Option>> optionsMap) {

        return "result";
    }
```

此外，使用 jQuery 的 Ajax 请求返回 json 化的所有 answer_info 表中的数据来渲染前端页面，需要 **@GetMapping** 和 **@ResponseBody** 注解配合使用，对应控制器方法为：

```java
    @CrossOrigin // 允许跨域请求(OCR)
    @GetMapping(path = "/results", produces = "application/json")
    @ResponseBody // 请求结果查找得到 Json 文件
    public Iterable<Answer> getResJson() {

        return answerRepo.findAll();
      
    }
```

 result 结果展示页面：

```html
<body>

<div class="container my-5">

    <div class="jumbotron border border-info">

        <h1 class="text-info">Result Page</h1>
        <table class="table table-hover" th:each="optionsForAQ : ${optionsMap}" th:id="'question' + ${optionsForAQ.key.id}" >

            <thead class="thead-dark">
            <tr>
                <th th:text="${optionsForAQ.key.question_name}" th:colspan="${optionsForAQ.value.size()}">QUESTION TITLE</th>
            </tr>
            </thead>

            <tbody>
            <tr>
                <th th:each="option : ${optionsForAQ.value}" th:text="${option.option_content}">OPTION</th>
            </tr>
            <tr>
                <td th:each="option : ${optionsForAQ.value}" th:id="'option' + ${option.id}">0</td>
            </tr>
            </tbody>

        </table>

    </div>

</div>

</body>

<script type="text/javascript">

    $(document).ready(function() {
        $.ajax({
            type: "GET",
            url: "http://localhost:8080/results",          
            dataType: "json",
            success: function (data) {
                let option_id = 0;
                let voteNum = 0;
                // 读取id -> 读取表单值 -> 表单值增1
                $.each(data, function (index, answer) {
                    option_id = answer["option_id"];
                    voteNum = $("#option" + option_id).text();              
                    $("#option" + option_id).text(parseInt(voteNum) + 1);
                });
            },
            error: function (result) {
                alert(result.responseText);
            }
        });
    });

</script>
```

值得一提的是此页面使用 thymeleaf 的 **th:colspan** 来动态定义表头跨距，使用 jQuery 读取 json 中的每一条记录并操作页面元素，最终得到这样的结果图：

![截屏2020-08-28 15.35.17](https://tva1.sinaimg.cn/large/007S8ZIlgy1gi7stkj4qlj31570u0gq4.jpg)

# 小结

到这里，我们得到了一个可以运行（如果注入 Spring Security 的话需要先使用控制台的密码登录，用户名默认为 user）的、基本实现了 **展示问卷-->投票-->提交-->展示结果页面** 的 Spring app，下一阶段我们将使用 Spring Security 来实现用户注册、登录、认证等功能。

