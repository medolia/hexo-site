---
title: mac os 编译 openjdk14 源码
date: 2020/9/1 14:26:15
categories: 编程
tags: 
  - openjdk
excerpt: 在 mac os 操作系统下完成 openjdk 的源码编译并导入至 CLion 中
---



# 前言

想要窥探 Java 虚拟机内部的实现原理，最直接的一条路径就是编译一套自己的 JDK，通过阅读和跟踪调试 JDK 源码来了解 Java 技术体系的运作是最快、最贴近本质也可能是最有挑战的做法，这里选择使用较为广泛的 OpenJDK [官网链接](http://openjdk.java.net/) 。

# 搭建环境

## 获取源码

这里选择使用 openjdk14，进入官网的 [下载页面](http://hg.openjdk.java.net/jdk/jdk14) ，点击左边栏的 **zip** 下载源码压缩包（大小大概 200MB）。

![截屏2020-09-01 20.54.14](https://tva1.sinaimg.cn/large/007S8ZIlgy1gibg0xse8dj314y0u07qo.jpg)

下载完成后解压至本地即可。

## Boot JDK

默认已经安装好 [Homebrew](https://brew.sh/) ，编译 JDK 需要已有版本至少为 N-1 的 JDK，比如我们选择安装 openjdk14 那么需要系统 `java -version` 显示为 **13** 或 **14** 的 JDK。安装 JDK 可以参考这篇 [教程](https://www.cnblogs.com/imzhizi/p/macos-jdk-installation-homebrew.html) ，切换 JDK 可以参考这篇 [短文](https://blog.csdn.net/weixin_42112888/article/details/82851594) 。

## 其他依赖

+ Xcode: 主要提供 CLang 编译器
+ ccache: 提供编译器缓存
+ freetype
+ autoconf

```sh
brew install ccache freetype autoconf
```

## 检测依赖

命令行 `cd [jdk 文件夹]` 进入源码文件夹，运行 `bash ./configure` 检测依赖，可根据提示补充缺少的依赖项。

**常见报错**

```
xcode-select: error: tool 'xcodebuild' requires Xcode,
but active developer directory
'/Library/Developer/CommandLineTools' is a command line tools instance
```

**解决方案**

`sudo xcode-select --switch /Applications/Xcode.app/Contents/Developer`

如果出现以下信息就说明配置成功了：

![截屏2020-09-01 21.15.47](https://tva1.sinaimg.cn/large/007S8ZIlgy1gibgnc55t9j310a0o2qaj.jpg)

# 编译

## 编译参数

可以使用 `bash configure [option]` 来设置编译参数， `bash configure --help` 可以查询到所有可用的参数，这里介绍几个常用的：

+ `--with-debug-level=slowdebug`: 设置编译级别为 slowdebug，是携带最多调试信息的模式
+ `--with-jvm-variants=server`: 编译 server 模式的虚拟机
+ `--with-target-bits=64`: 设置 64 位编译
+ `--enable-ccache`: 启用 ccache
+ `--with-num-cores`=6: 使用 6 个 CPU核心 
+ `--with-memory-size=8000`: 使用 8GB 内存
+ `--disable-warnings-as-errors`: 忽略警告，减少报错

```
% bash configure --with-debug-level=slowdebug --with-jvm-variants=server --with-target-bits=64 --enable-ccache --with-num-cores=6 --with-memory-size=8000 --disable-warnings-as-errors      
```

值得一提的是在最终确定编译参数后，需要使用 `make clean` 和 `make dist-clean` 清除之前的配置文件，然后再运行最终的编译参数命令。

## 开始编译

`make images` 即可开始编译，正常等待 10 分钟左右即可完成编译：

![img](https://tva1.sinaimg.cn/large/007S8ZIlgy1gibie3v4qzj31ek0lc12r.jpg)

**验证**

导航至 build 文件夹内的 bin，使用`./java -version` 验证 JDK 版本：

![截屏2020-09-01 22.23.07](https://tva1.sinaimg.cn/large/007S8ZIlgy1gibilfg3pyj311w0qinal.jpg)

# 导入项目至 CLion

打开 CLion，选择 **Open CMake Project from Sources**，选择整个 JDK 文件夹，之后勾选项默认，直接点击 **OK** 即可。

## 编辑配置

> Run -> Edit Configurations

![截屏2020-09-01 22.34.27](https://tva1.sinaimg.cn/large/007S8ZIlgy1gibixnbzp3j31910u0to9.jpg)

+ Executable: 之前 bin 文件夹内的名为 **java**  的执行文件

+ Program Arguments: **-version** 输出 Java 版本
+ 删除 Before launch 里的 **Build** 防止报错

好了，可以愉快的开始读源码了。



