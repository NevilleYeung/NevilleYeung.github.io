---
layout: post
title: "ServiceComb代码分析——Java Chassis"
date: 2018-07-15 
tags: ServiceComb 代码分析
---

ServiceComb是华为开源的微服务框架，目前已发布第一个Apache孵化版本，官网：http://servicecomb.incubator.apache.org/cn/。本文是基于1.0.0-mX版本的代码进行分析。       
主要模块有Java Chassis、服务中心和Saga，下文是对Java Chassis的代码走读和分析。
 

### bmi微服务实例       

根据官网的快速入门，我们调试了samples目录下的bmi微服务实例。    
启动入口如下：
```     
@SpringBootApplication
@EnableServiceComb
public class CalculatorApplication {

  public static void main(String[] args) {
    SpringApplication.run(CalculatorApplication.class, args);
  }
}
```  



<br>

转载请注明：[蚊帐的博客](https://nevilleyeung.github.io) » [点击阅读原文](https://nevilleyeung.github.io/2018/07/Details-ServiceComb-CodeAnalysis-JavaChassis/) 

 



