---
layout: mypost
title: Homebrew忽略依赖安装
categories: [环境配置]
---

### 问题描述与解决办法

在使用 Homebrew 下载时，一般会自动下载相关依赖，并且这些依赖是最新版本的。比如在下载 Maven 时，会自动下载最新的 JDK。

以 Maven 为例，可以使用下面的命令忽略依赖安装：

```bash
brew install --ignore-dependencies maven
```