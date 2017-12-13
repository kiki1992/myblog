---
title: Spring设计模式应用之适配器模式
date: 2017-12-06 18:10:32
summary: 本篇通过Spring中使用适配器模式进行程序设计的例子，简单分析了适配器设计模式在Spring中的应用。
tags: [Spring Framework,设计模式]
---
# Spring设计模式应用之适配器模式
## 适配器模式
> 简单来说，适配器模式的设计目的就是为了让那些原本因为接口不兼容而无法在一起工作的类能通过适配器这个中间人的协调而一起工作。

## Spring中应用适配器模式的例子
> Spring中应用适配器模式的地方挺多的，这里只是挑了一个例子做简单分析，看看这些著名框架中都是怎么运用设计模式来提高程序的通用性和可扩展性的。

***

#### AdvisorAdapter
使用过Spring AOP功能的同学想必对Advisor不会陌生，本篇举的例子就是Advisor的适配器类AdvisorAdapter。
首先看一下AdvisorAdapter是怎么定义的。
```java
public interface AdvisorAdapter {
	boolean supportsAdvice(Advice advice);
	MethodInterceptor getInterceptor(Advisor advisor);
}
```

从上面的代码可以看到，AdvisorAdapter是一个接口类，定义了以下两个方法:

* 1.supportsAdvice(Advice advice) -- 用于检查当前适配器是否支持指定advice
* 2.getInterceptor(Advisor advisor) -- 根据指定advisor获取AOP方法拦截器，调用的前提当然是第一个方法返回true

#### AdvisorAdapter实现类
Spring AOP中针对AdvisorAdapter接口有多个实现类，这里贴两个实现的例子。
```java
// 方法前置通知适配器
class MethodBeforeAdviceAdapter implements AdvisorAdapter, Serializable {

	@Override
	public boolean supportsAdvice(Advice advice) {
		return (advice instanceof MethodBeforeAdvice);
	}

	@Override
	public MethodInterceptor getInterceptor(Advisor advisor) {
		MethodBeforeAdvice advice = (MethodBeforeAdvice) advisor.getAdvice();
		return new MethodBeforeAdviceInterceptor(advice);
	}

}
```

```java
// 方法后置通知适配器
class AfterReturningAdviceAdapter implements AdvisorAdapter, Serializable {

	@Override
	public boolean supportsAdvice(Advice advice) {
		return (advice instanceof AfterReturningAdvice);
	}

	@Override
	public MethodInterceptor getInterceptor(Advisor advisor) {
		AfterReturningAdvice advice = (AfterReturningAdvice) advisor.getAdvice();
		return new AfterReturningAdviceInterceptor(advice);
	}

}
```

#### 适配器的使用
上面提到的Advisor适配器实现类会由DefaultAdvisorAdapterRegistry类自动注册，然后在类的其它方法被调用时发挥作用，具体看以下代码。

首先是适配器实现类的注册。
可以看到，在生成DefaultAdvisorAdapterRegistry实例时会默认注册上述适配器实现类。
```java
public DefaultAdvisorAdapterRegistry() {
	registerAdvisorAdapter(new MethodBeforeAdviceAdapter());
	registerAdvisorAdapter(new AfterReturningAdviceAdapter());
	registerAdvisorAdapter(new ThrowsAdviceAdapter());
}
```
接下来是使用这些适配器实现类的地方。
方法会遍历适配器实现类列表，检查适配器是否支持当前通知，如果支持再调用其获取拦截器的方法获得对应的拦截器。因此在使用方法时我们无需关注具体使用了那个实现类，这也是代码良好通用性的体现。
```java
public MethodInterceptor[] getInterceptors(Advisor advisor) throws UnknownAdviceTypeException {
	List<MethodInterceptor> interceptors = new ArrayList<MethodInterceptor>(3);
	Advice advice = advisor.getAdvice();
	if (advice instanceof MethodInterceptor) {
		interceptors.add((MethodInterceptor) advice);
	}
	// 遍历适配器实现类寻找支持当前通知的那一个
	for (AdvisorAdapter adapter : this.adapters) {
		if (adapter.supportsAdvice(advice)) {
			interceptors.add(adapter.getInterceptor(advisor));
		}
	}
	if (interceptors.isEmpty()) {
		throw new UnknownAdviceTypeException(advisor.getAdvice());
	}
	return interceptors.toArray(new MethodInterceptor[interceptors.size()]);
}
```
另外我们还注意到，DefaultAdvisorAdapterRegistry类有一个registerAdvisorAdapter方法，也就是支持使用者自定义和注册自己的适配器实现类，而无需对现有代码做任何修改，这也体现了设计模式在提高代码可扩展性方面的应用。
```java
public void registerAdvisorAdapter(AdvisorAdapter adapter) {
	this.adapters.add(adapter);
}
```

以上就是对Spring中应用适配器设计模式的一个简单介绍。

