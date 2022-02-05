+++
title = "通过编程配置log4j2"
date = 2021-11-06T00:23:14+08:00
draft = false
archives = 2021
categories = "programming"
tags = ["log4j2","java"]
+++

最近在使用Log4j2的时候，碰到了一些问题：无论是使用`Configurator.initialize`还是`LogManager.getContext().reconfigure()`都无法完全更新log4j2的配置。

在查看了Log4j2的源代码过后，笔者找到了解决方案。

<!--more-->

tl;dr.
```java
Configurator.initialize(
    ClassLoader.getSystemClassLoader(),
    configuration
);

Configurator.initialize(
    configuration
);
// 缺一不可
```

如果观察`LogManager.getLogger(Class<?>)`，会发现其调用的是`getContext(cls.getClassLoader(), false).getLogger(cls);`。
```java
public static Logger getLogger(final Class<?> clazz) {
    final Class<?> cls = callerClass(clazz);
    // note this:     ↓
    return getContext(cls.getClassLoader(), false).getLogger(cls);
}
```
对比之下的`LogManager.getLogger(String)`
```java
public static Logger getLogger(final String name) {
        // note this:         ↓
        return name != null ? getContext(false).getLogger(name) : getLogger(StackLocatorUtil.getCallerClass(2));
    }
```

如果使用`LogManager.getLogger(String)`这个函数，那么loader将会是null(对比之下，`LogManager.getLogger(Class<?>)`使用`cls.getClassLoader()`获取了loader)。可能会有一条可疑的警告信息:
`WARNING: sun.reflect.Reflection.getCallerClass is not supported. This will impact performance.`
并且会获取到与带有loader参数不同的`LoggerContext`。

`getLogger(Class<?>)`虽然一直作为log4j2的典型用法，仍然有一些第三方库使用`getLogger(String)`。比如netty。

所需需要对带classLoader参数和不带此参数的`LoggerContext`都进行更新。代码见上。
