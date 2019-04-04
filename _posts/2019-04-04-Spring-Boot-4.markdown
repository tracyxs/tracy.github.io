---
layout: post
title: "Spring Boot (4) -日志"
date: 2019-04-04 14:34
categories: java springboot
display: true
comments: true
---

  Spring Boot 默认会输出 INFO级别的日志，还包括一些相关的启动详细信息。如果启动失败，Spring Boot的FailureAnalyzers就会输出相关的错误信息，然后一个专用的处理器会对这个错误进行相关的处理。Spring Boot 提供了很多这样的失败分析器，如果还不能满足要求，你也可以自己通过实现FailureAnalyzer接口来构建自己的错误分析器。

  1、Sprint Boot的日志级别的配置方式有两种：
  一种是使用Java -jar命令来启动项目，那么你可以通过如下方式来设置debug属性：

    $ java -jar myproject-0.0.1-SNAPSHOT.jar —debug

  另外一种是修改配置文件application.properties中进行配置完成日志记录的级别控制。日志级别LEVEL选项包括TRACE, DEBUG, INFO, WARN, ERROR, FATAL, OFF

  配置格式：logging.level.*=LEVEL

  举例：logging.level.com.zmcc=DEBUG：com.zmcc包下所有的class以DEBUG级别输出

  logging.level.root=WARN：root日志以WARN级别输出

  2、输出日志到文件：Spring Boot默认配置只会输出到控制台，并不会记录到文件中，但是我们通常生产环境使用时都需要以文件方式记录。若要增加文件输出，需要在application.properties中配置logging.file或logging.path属性。

  logging.file，设置文件，可以是绝对路径，也可以是相对路径。如：logging.file=my.log

  logging.path，设置目录，会在该目录下创建spring.log文件，并写入日志内容，如：logging.path=/var/log

  日志文件会在10Mb大小的时候被截断，产生新的日志文件，默认级别为：ERROR、WARN、INFO











  


  
