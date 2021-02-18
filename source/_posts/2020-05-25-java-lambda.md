---
title: Java lambda 表达式浅析
date: 2020/5/25 14:26:15
categories: 编程
tags: 
  - java
  - lambda 表达式
excerpt: 个人对 jdk 1.8 新增的 lambda 表达式的一些理解
---


# 前言

我们以一段代码为引子

```java

import java.awt.*;
import java.awt.event.*;
import java.time.*;
import javax.swing.*;

public class TimerTest {
    public static void main(String[] args) {
        var listener = new TimePrinter();

        // construct a timer that calls the listener
        // once every second
        var timer = new Timer(1000, listener);
        timer.start();

        // keep program running until the user selects "OK"
        JOptionPane.showMessageDialog(null, "Quit program?");
        System.exit(0);
    }
}

class TimePrinter implements ActionListener {
    public void actionPerformed(ActionEvent event) {
        System.out.println("the time is "+ Instant.ofEpochMilli(event.getWhen()));
        Toolkit.getDefaultToolkit().beep();
    }
}

```
这是一个很简单的 timer 小程序，功能是每过一秒报时一次并发出一声"beep"，直至点击窗口中的 "Quit program" 按钮。

但仔细想想，在这过程中我们先创造了一个实现了 ActionListener 接口的 TimePrinter 内部类，接着又创造了一个内部类实例传入 Timer 的构造器中。

这期间新增了很多也许在一个程序中只会引用一次的变量名，况且我们发现 ActionListener 接口只有一个抽象方法 actionPerformed，也许有一种特性可以自动识别这样仅含一个抽象方法的特殊接口，我们只需要指定变量名和实现这个唯一方法就可以了。

# lambda 表达式

```java

import java.awt.*;
import java.time.*;
import javax.swing.*;

public class TimerTest {
    public static void main(String[] args) {

        var timer = new Timer(1000, event -> {
            System.out.println("At the tone, the time is " +
                    Instant.ofEpochMilli(event.getWhen()));
            Toolkit.getDefaultToolkit().beep();
        });

        timer.start();

        JOptionPane.showMessageDialog(null, "Quit program?");
        System.exit(0);
    }
}

```
与上一个程序的功能一摸一样，但这次不再需要创造新内部类，新增变量名等操作，专注于 Timer 构造器中的 lambda 代码，发现我们只需定义抽象方法的参数名和实现方法即可——JVM 会自动识别 lambda 表达式对应 ActionListener 中唯一的抽象方法 actionPerformed。 

像 ActionListener 这样只有一个抽象方法的接口被称为**函数式接口**，只有这种接口可以接受 lambda 表达式（例: Object 类型就不行）。

# 方法引用

来看两段等效的代码：`var timer = new Timer(1000, event -> System.out.println(event));` 与 `var timer = new Timer(1000, System.out::println);`

当 lambda 表达式为不含"{...}"的简单表达时，也可以使用"::"**方法引用**，这将指示编译器生成一个函数式接口实例并覆盖那个唯一的抽象方法，此例中 actionPerformed(ActionEvent event) 将由 System.out.println(event) 实现。 

方法引用有三种情况
1. object::instanceMethod 方法引用等价于传递参数的 lambda 表达式，例：`System.out::println` 等价于 `x -> System.out.println(x)`
2. Class::instanceMethod 第一个参数会传入隐式参数，而其余（若存在）参数传入显式参数，例：`String::trim` 等价于`x -> x.trim()`，`String::concat` 等价于 `x -> x.concat(y)`
3. Class::staticMethod 全部参数传入静态方法，例：`Integer::sum` 等价于 `(x, y) -> Integer.sum(x, y)`

特别的，this::instanceMethod super::instanceMethod 对应此类和超类对象方法引用（第一种情况），Class::new 对应构造器引用（例：`Integer[]::new` 等价于 `n -> new Integer[n]`，n 为数组长度）
