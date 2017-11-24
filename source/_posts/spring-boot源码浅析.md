---
title: spring boot源码浅析--应用启动
date: 2017-11-24 15:56:32
summary: 本篇从源码的角度简单分析了SpringApplication的启动过程
tags: spring-boot 
font-size: 10px
---

#### SpringApplication的启动过程

```java
private void initialize(Object[] sources) {
	if (sources != null && sources.length > 0) {
		this.sources.addAll(Arrays.asList(sources));
	}
	this.webEnvironment = deduceWebEnvironment();
	setInitializers((Collection) getSpringFactoriesInstances(
			ApplicationContextInitializer.class));
	setListeners((Collection) getSpringFactoriesInstances(ApplicationListener.class));
	this.mainApplicationClass = deduceMainApplicationClass();
}
```
