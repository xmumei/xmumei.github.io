---
layout:mypost
title: Javascript Long 类型精度丢失
categories: [踩坑]
---

### 现象

后端生成纯数字的 UUID（共19位）返回给前端后发生精度丢失现象。具体表现为 `1915379012289929218` 变为 `1915379012289929200`。通过API测试得到的结果正常，定位到问题可能出现在前端。

### 原因

在 Javascript 中数字类型为双精度浮点数，支持的最大范围是 `2^53 - 1`，超过的部分直接截取变为零。

### 解决方法

使用 Jackson 的 `@JsonFormat` 注解在序列化时将 Long 转换成字符串类型。

```java
@JsonFormat(shape = JsonFormat.Shape.STRING)
private Long id;
```