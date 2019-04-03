---
layout: post
title: "Spring Boot (3) -pom.xml配置和工程启动"
date: 2019-04-03 10:28
categories: java springboot
display: true
comments: true
---

  pom.xml配置说明：

  1、  &lt;parent&gt;元素：指定了Spring Boot的父POM，并包含了常见组件的定义，不需要手动配置这些组件。

  &lt;parent&gt;
        &lt;groupId&gt;org.springframework.boot&lt;/groupId&gt;
		&lt;artifactId&gt;spring-boot-starter-parent&lt;/artifactId&gt;
		&lt;version&gt;2.0.1.RELEASE&lt;/version&gt;
  &lt;/parent&gt;

  2、&lt;dependency&gt;元素：指定了依赖的组件。如下配置说明该应用程序是一个WBE应用。

  &lt;dependency&gt;
		&lt;groupId&gt;org.springframework.boot&lt;/groupId&gt;
		&lt;artifactId&gt;spring-boot-starter-web&lt;/artifactId&gt;			
  &lt;/dependency&gt;

  3、&lt;build&gt;元素：指定编译插件。如下配置说明使用maven-war-plugin插件生成该应用程序。

  &lt;build&gt;
		&lt;plugins&gt;
			&lt;plugin&gt;
				&lt;groupId&gt;org.apache.maven.plugins&lt;/groupId&gt;
				&lt;artifactId&gt;maven-war-plugin&lt;/artifactId&gt;
			&lt;/plugin&gt;
		&lt;/plugins&gt;
  &lt;/build&gt;

  4、&lt;exclusions&gt;元素：用于依赖排除。
  举个例子，tomcat是默认的嵌入式web服务器容器，引入spring-boot-starter-web时就默认引入了spring-boot-starter-tomcat。若我们不想使用tomcat，而想使用jetty，需要修改&lt;dependency&gt;，使用排除&lt;exclusions&gt;元素，并引入jetty依赖。如下：

  &lt;dependencies&gt;
    &lt;dependency&gt;
        &lt;groupId&gt;org.springframework.boot&lt;/groupId&gt;
        &lt;artifactId&gt;spring-boot-starter-web&lt;/artifactId&gt;
        &lt;exclusions&gt;
            &lt;exclusion&gt;
                &lt;groupId&gt;org.springframework.boot&lt;/groupId&gt;
                &lt;artifactId&gt;spring-boot-starter-tomcat&lt;/artifactId&gt;
            &lt;/exclusion&gt;
        &lt;/exclusions&gt;
    &lt;/dependency&gt;
    &lt;dependency&gt;
        &lt;groupId&gt;org.springframework.boot&lt;/groupId&gt;
        &lt;artifactId&gt;spring-boot-starter-jetty&lt;/artifactId&gt;
    &lt;/dependency&gt;
&lt;/dependencies&gt;
 
  那如何使用maven构建工程？方法很多，可以从IDE，比如eclipse中运行maven构建：请右键单击 POM 文件并选择 Run As &gt; Maven Build。在 Goals 文本字段中，输入 clean 和 package，然后单击 Run 按钮。也可以从命令行运行maven构建：打开命令行界面，进入工程目录，执行mvn clean package即可。

  然后就是运行工程了。要运行刚创建的可执行JAR，请打开命令行界面，进入工程目录，然后执行：
  java -jar target/工程名-1.0-SNAPSHOT.jar。
  或者在如eclipse中找到主程序application.java,右键run as applications即可。
 




  


  
