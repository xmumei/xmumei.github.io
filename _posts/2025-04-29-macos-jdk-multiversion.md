---
layout: mypost
title: MacOS配置多版本JDK
categories: [环境配置]
---

### 下载安装

>   推荐从各个发布商的官网下载 JDK，比如 [Oracle JDK](https://www.oracle.com/sg/java/technologies/downloads)，[Zulu JDK](https://www.azul.com/downloads)。
>
>   如果你对发布商版本有疑惑，或者在寻找对 ARM 架构支持更好的发行版，可以参考文末的内容。

下载后安装即可，只要是 `.pkg` 文件，安装后都在 `/Library/Java/JavaVirtualMachine/` 目录下。

### 配置环境

Sonoma（也就是 macOS 14）默认的终端 shell 为 zsh，配置文件名为 `.zshrc`。如果你在使用其他的系统版本，请确定好自己的终端 shell。

可以通过下面的命令打开配置文件：

```bash
vim ~/.zshrc
```

下面使用 JDK8 和 JDK17 为例，添加如下配置（请根据实际安装的版本进行路径调整）：

```bash
# Java
JAVA_8_HOME=/Library/Java/JavaVirtualMachines/zulu-8.jdk/Contents/Home
JAVA_17_HOME=/Library/Java/JavaVirtualMachines/zulu-17.jdk/Contents/Home
 
alias jdk8="export JAVA_HOME=$JAVA_8_HOME"
alias jdk17="export JAVA_HOME=$JAVA_17_HOME"
 
export JAVA_HOME=$JAVA_17_HOME
export PATH="$JAVA_HOME:$PATH"
```

保存并退出后，输入下面的命令更新配置文件。

```bash
source ~/.zshrc
```

### 验证

在命令行输入 `jdk8` 和 `jdk17` 能够切换默认 JDK 版本即可。可以使用下面的命令查看当前默认的 JDK 版本：

```bash
java -version
```

### 我该选择哪个JDK版本

>   不同发布商提供的 JDK 在性能、平台支持上存在差异。如果你不确定选择哪一种，可以参考[这个网站](https://whichjdk.com/)给出的建议。

macOS 对 ARM 架构（即 AArch64）的官方支持是从 OpenJDK 17 开始的，对应的提案为 [JEP 391](https://openjdk.java.net/jeps/391)。也就是说大部分发布商对于 JDK17+ 的版本都提供了原生 ARM 版本的构建。

在 JDK17 之前除了少数几个发布商提供了原生支持，其他的都是通过运行基于 x64 架构的构建版本，依赖 macOS 的 Rosetta 2 转译实现的。虽然运行稳定，但性能上有明显下降。

提供了旧版本（JDK8、JDK11）原生支持的发布商有：

-   BellSoft
-   Amazon Corretto
-   Azul Zulu



