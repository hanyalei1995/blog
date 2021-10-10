---
title: spring boot 自定义注解
date: 2021-09-30 17:14:08
description: spring boot 自定义注解的实现
tags:
 - spring boot
categories:
 - spring boot
---

# 实现spring boot 自定义注解有关的注解

@Documented  @Retention  @Target

@Documented注解作用是是否可以被类似javadoc的文档工具文档化, 没有成员取值, 仅仅标记作用

@Retention注解的作用是 注解的生命周期, 即标识该注解的有效范围(), 该注解有3个成员取值, 分别是:

1. SOURCE:在源文件中有效（即源文件保留）
2. CLASS:在class文件中有效（即class保留）
3. RUNTIME:在运行时有效（即运行时保留）

@Target注解的作用是注解的使用范围, 即标识该注解可以用在哪里, 成员取值分别为:

1. @Target(ElementType.TYPE)——接口、类、枚举、注解
2. @Target(ElementType.FIELD)——字段、枚举的常量
3. @Target(ElementType.METHOD)——方法
4. @Target(ElementType.PARAMETER)——方法参数
5. @Target(ElementType.CONSTRUCTOR) ——构造函数
6. @Target(ElementType.LOCAL_VARIABLE)——局部变量
7. @Target(ElementType.ANNOTATION_TYPE)——注解
8. @Target(ElementType.PACKAGE)——包

实现demo

```java
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
public @interface Log {
   String value() default "";
}
```

