---
title: Android性能优化之使用AOP结合Systrace查找性能短板
date: 2018-08-19 15:45:22
tags: Android
---
# 0x01 什么是AOP？怎么使用Systrace？
AOP意即面向切面编程，通过预编译方式和运行期动态代理实现程序功能的统一维护的一种技术。
我在Android上使用AOP主要是为了做性能检测，Systrace默认没有针对所有方法进行跟踪，如果需要针对特殊方法进行跟踪，那么需要在java代码里面插入begin和endTrace等方法。这样不仅破坏了原有代码结构，而且难以对所有方法做追踪。
AspectJ做到了不侵入代码就可以插入追踪代码。
AOP有几个关键词：
1. Join Points即是程序运行的一个执行点，包括函数执行、函数调用、构造方法调用等等都是一个Join Points。
2. Advice Advice就是我们插入的代码以何种方式插入，有Before还有After、Around。
3. 自定义Pointcuts可以让我们更加精确的切入一个或多个指定的切入点。

# 0x02 准备工作
首先需要在项目工程下引入AspectJ