---
title: BeanPostProcessor浅析
date: 2018-01-11 11:16:32
summary: 本篇对BeanPostProcessor接口及部分实现做了简单的分析记录。
tags: Spring Framework
---

## BeanPostProcessor浅析

### 概要

> * BeanPostProcessor接口定义了postProcessBeforeInitialization和postProcessAfterInitialization两个方法，允许我们对实例化Bean做一些自定义的修改。
> * ApplicationContexts可以自动检测实现了BeanPostProcessor接口的Bean并将其自定义方法适用到随后创建的任何Bean。
> * 使用BeanFactory也可以设置BeanPostProcessors，比如你可以调用ConfigurableBeanFactory的addBeanPostProcessor方法添加一个BeanPostProcessor实现，这样一来该工厂创建的Bean都会适用BeanPostProcessor实现类的方法。



### 接口定义

BeanPostProcessor接口定义了postProcessBeforeInitialization和postProcessAfterInitialization两个方法，下面先简单介绍一下这两个方法分别是干什么的。

***postProcessBeforeInitialization***

如果需要在Bean完成初始化之后，Bean初始化方法回调之前对新创建的Bean做自定义修改，就可以实现postProcessBeforeInitialization方法。在本方法调用之前Bean的属性值已经完成设置，而本方法会先于任何Bean初始化回调执行，比如InitializingBean接口定义的afterPropertiesSet方法/自定义的init-method方法。

**需要注意的一点是：**如果本方法返回null，那么后续其他BeanPostProcessors不会被使用。

***postProcessAfterInitialization***

如果需要在Bean初始化方法回调执行之后对Bean做自定义修改，则需要实现postProcessAfterInitialization方法。

当新创建的Bean是FactoryBean时，本方法会同时作用于FactoryBean以及由该FactoryBean创建的Bean，可以在本方法通过判断入参bean instanceof FactoryBean来决定适用本方法的规则。

和postProcessBeforeInitialization方法一样，如果返回null，那么后续其他BeanPostProcessors不会被使用。

```java
public interface BeanPostProcessor {
  
   Object postProcessBeforeInitialization(Object bean, String beanName) throws BeansException;
  
   Object postProcessAfterInitialization(Object bean, String beanName) throws BeansException;

}
```



### 使用实例

简单介绍了BeanPostProcessor接口的一些特性之后，让我们通过一些实例来看一下BeanPostProcessor的具体应用吧。

首先来看一个Spring框架实现postProcessBeforeInitialization方法的例子 -- ApplicationContextAwareProcessor类。

***ApplicationContextAwareProcessor***

这个类的作用是给实现某些特定Aware相关接口的类自动设置ApplicationContext以及从ApplicationContext获取的Environment，BeanFactory。考虑到Bean的初始化回调方法中也可能会利用到ApplicationContext来获取上下文信息，重写BeanPostProcessor的postProcessBeforeInitialization方法是一个合理的选择。***对于这一点，BeanPostProcessor的官方文档中也提到，如果是根据标记接口做一些处理，那么一般选择实现postProcessBeforeInitialization接口。***

查看ApplicationContextAwareProcessor中对postProcessBeforeInitialization的实现可以发现，除去一些安全相关的处理，方法最终调用了处理invokeAwareInterfaces来处理ApplicationContext的自动设置。而处理的方式也相当简单。

```java
public Object postProcessBeforeInitialization(final Object bean, String beanName) throws BeansException {
   AccessControlContext acc = null;

   if (System.getSecurityManager() != null &&
         (bean instanceof EnvironmentAware || bean instanceof EmbeddedValueResolverAware ||
               bean instanceof ResourceLoaderAware || bean instanceof ApplicationEventPublisherAware ||
               bean instanceof MessageSourceAware || bean instanceof ApplicationContextAware)) {
      acc = this.applicationContext.getBeanFactory().getAccessControlContext();
   }

   if (acc != null) {
      AccessController.doPrivileged(new PrivilegedAction<Object>() {
         @Override
         public Object run() {
            invokeAwareInterfaces(bean);
            return null;
         }
      }, acc);
   }
   else {
      invokeAwareInterfaces(bean);
   }

   return bean;
}
```

可以看到，invokeAwareInterfaces会根据传入Bean具体实现的接口分别调用类内部对应的set方法来完成ApplicationContext的设置。

```java
private void invokeAwareInterfaces(Object bean) {
   if (bean instanceof Aware) {
      if (bean instanceof EnvironmentAware) {
         ((EnvironmentAware) bean).setEnvironment(this.applicationContext.getEnvironment());
      }
      if (bean instanceof EmbeddedValueResolverAware) {
         ((EmbeddedValueResolverAware) bean).setEmbeddedValueResolver(
               new EmbeddedValueResolver(this.applicationContext.getBeanFactory()));
      }
      if (bean instanceof ResourceLoaderAware) {
         ((ResourceLoaderAware) bean).setResourceLoader(this.applicationContext);
      }
      if (bean instanceof ApplicationEventPublisherAware) {
         ((ApplicationEventPublisherAware) bean).setApplicationEventPublisher(this.applicationContext);
      }
      if (bean instanceof MessageSourceAware) {
         ((MessageSourceAware) bean).setMessageSource(this.applicationContext);
      }
      if (bean instanceof ApplicationContextAware) {
         ((ApplicationContextAware) bean).setApplicationContext(this.applicationContext);
      }
   }
}
```

下面来看一个Spring框架实现postProcessAfterInitialization方法的例子 -- AbstractAutoProxyCreator类。

***AbstractAutoProxyCreator***

AbstractAutoProxyCreator类实现了postProcessAfterInitialization方法为特定的Bean设置代理。***BeanPostProcessor的官方文档中提到，如果需要为Bean设置代理，那么一般选择实现postProcessAfterInitialization接口。***

下面是postProcessAfterInitialization方法的实现。

```java
@Override
public Object postProcessAfterInitialization(Object bean, String beanName) throws BeansException {
   if (bean != null) {
      Object cacheKey = getCacheKey(bean.getClass(), beanName);
      if (!this.earlyProxyReferences.contains(cacheKey)) {
         return wrapIfNecessary(bean, beanName, cacheKey); // 为bean设置代理
      }
   }
   return bean;
}
```