---
layout: post
title:  "Java Performance: The Definitive Guide(1st)"
date:   2015-02-24 12:16:58
category: java  
tags: [java,performance,book]  
author: liyan  
image: image/larg.png  
---

## 目录

- 1章 介绍
- 2章 常用的测试java应用的方法以及经常遇到的陷阱
- 3章 java应用常用的监控工具
- 4章 JIT编译器
- 5&6章 GC
- 7章 jvm堆内存
- 8章 原生内存
- 9章 线程性能
- 10章 JavaEE API
- 11章 JPA和JDBC
- 12章 JavaSE API建议
- 附录A 所有的调优参数注解

---

## 1章
<!--break-->
### 性能story

- #### 算法尽可能高效

- #### 代码尽可能少

- #### 过早优化prematurely optimize

> [Premature optimization][1] is the root of all evil --- Donald Knuth

- #### 关注外部资源的影响：例如数据库经常是性能的瓶颈

> 如果一个java应用是完全独立的没有使用任何的外部资源，那么应用的性能只和java应用本身有关.
  一旦使用了外部资源（例如：数据库），那么数据库应用和Java应用本身都对性能有重要影响。
  特别是在分布式应用环境中，有Java EE 应用服务器、负载均衡服务器、数据库以及后端企业信息系统等，这种情况下Java应用服务器对整体性能的影响可能是最小的。
  这本书不讲有关系统整体性能的内容，在这种环境下，一整套监控必须运用到系统的各个部分上，包括对CPU使用率、I/O延迟以及吞吐量的度量和分析，只有这样才能找到具体是哪个部分影响了整个系统的性能，换句话说找到系统性能的瓶颈所在。

- #### 优化通用的case

> 优化代码中最费时的操作，但不是只关注和优化最底层的方法
  新代码出现性能问题很可能是机器的配置问题，反过来说很可能是JVM或操作系统的配置问题导致的，优先考虑最可能的情况而不是跳过直接去考虑不常见的情况
  对于错误率，如果用户能够接受10%的错误遗留能保证高性能，则不需要优化到1%反而使性能变差，要做出这种权衡。

[1]: http://c2.com/cgi/wiki?PrematureOptimization