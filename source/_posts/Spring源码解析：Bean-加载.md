---
title: Spring源码解析：Bean 加载
date: 2022-11-16 21:16:46
tags: Spring源码解析
categories: Spring
description: bean 初始化概述。
---

在前面几篇文章中，已经介绍了资源加载、BeanDefinition 加载、自定义标签，现在已经有了 BeanDefinition，接下来就是 bean 对象的实例化。

深入 spring 源码，加载 bean 对象其实是从一个 `doGetBean(String name, Class<T> requiredType, Object[] args, boolean typeCheckOnly)` 方法开始。

```java
```

