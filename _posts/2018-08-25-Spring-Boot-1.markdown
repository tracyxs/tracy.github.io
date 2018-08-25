---
layout: post
title: "Spring Boot (1) -启动引导类"
date: 2018-08-25 15:16
categories: java springboot
display: true
comments: true
---

  Spring Boot 是一个轻量级框架，可以完成基于 Spring 的应用程序的大部分配置工作。Spring Boot 的目的是提供一组工具，以便快速构建容易配置的 Spring 应用程序。
  Spring Boot 的默认启动引导类是Application.java。在Application的头部加上@SpringBootApplication注解，即开启组件扫描和自动配置。@SpringBootApplication注解是以下三个注解的组合：@Configuration，@ConponentScan，@EnableAutoConfiguration。其中，@Configuratio表示这个启动类是配置类，即配置到Spring容器中，@ConponentScan表示开启自动扫描，即在 spring 中启动组件扫描，默认扫描与配置类相同的包，@EnableAutoConfiguration表示自动装配，将所有符合自动配置条件的bean定义加载到IoC容器中。Application.java的main方法中，SpringApplication.run() 启动引导应用程序。
  那整个Spring Boot Web项目的启动过程是怎么样的呢？下节说明。

