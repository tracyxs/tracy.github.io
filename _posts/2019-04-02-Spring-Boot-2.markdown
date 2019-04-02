---
layout: post
title: "Spring Boot (2) -Spring Boot Starter"
date: 2019-04-02 16:23
categories: java springboot
display: true
comments: true
---

  Starter 是 Spring Boot 的一个重要组成部分，用于限制要执行的手动配置依赖项数量。实际上，Starter就是一组依赖项（比如 Maven POM），这些依赖项是Starter 所表示的应用程序类型所独有的。

  Starter的命名规则：一般都约定为spring-boot-starter-xyz,其中xyz就是要构建的应用程序类型。以下是一些常见的Spring Boot Starter：
  (1)spring-boot-starter-web 用于构建 RESTful Web 服务，它使用Tomcat 作为嵌入式应用程序容器,使用hibernate进行对象-关系映射（ORM),使用Apache Jackson绑定Json，使用Spring MVC作为Rest框架。

  (2)spring-boot-starter-jersey 是 spring-boot-starter-web 的一个替代，它使用 Apache Jersey 而不是 Spring MVC。
  
  (3)spring-boot-starter-jdbc 用于建立 JDBC 连接池。它基于 Tomcat 的 JDBC 连接池实现。


  
