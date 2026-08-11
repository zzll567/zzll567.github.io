---
title: SpringBoot Monitor服务监控
description: 
date: 2026-05-20 18:33:00 +0800
categories: [Blogging]
tags: [course]
pin: true
math: true
mermaid: true
---

与SpringBoot Admin不同的是，无需重新部署一个应用作为Admin Server，比较适用于单机SpringBoot服务监控。
来源：[[https://www.pomit.cn/SpringBootMonitor/]]

本文使用环境
JDK 1.8
Maven 3.5.4
SpringBoot 2.4.5

使用样例：
![示意图](/assets/img/2026-05-20/Pasted image 20260520154512.png)
![示意图](/assets/img/2026-05-20/Pasted image 20260520154533.png)

使用步骤：
##### 1.依赖
由于是基于SpringBoot Actuator的可视化页面，必须依赖于SpringBoot Actuator
```xml
<dependency>  
    <groupId>org.springframework.boot</groupId>  
    <artifactId>spring-boot-starter-actuator</artifactId>  
</dependency>
```

导入SpringBoot Monitor依赖
```xml
<dependency>  
    <groupId>cn.pomit</groupId>  
    <artifactId>spring-boot-monitor</artifactId>  
    <version>0.0.4</version>  
</dependency>
```
##### 2.配置
同样，因为使用了actuator，必须加上actuator的配置，开放endpoints
```yml
management:  
  endpoints:  
    web:  
      exposure:  
        include: "*"
```
##### 3.访问
如果当前的应用地址为[[http://127.0.0.1:8080]], spring-boot-monitor的访问地址为：[[http://127.0.0.1:8080/monitor]]

注意：需要在服务中放行 /monitor/** 路径