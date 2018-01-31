#### 自定义ComponentScan扫描规则

默认情况下，ComponentScan会将基础包中带@Component, @Repository, @Service, @Controller以及自定义Component注解的类作为候选组件。如果需要自定义ComponentScan的扫描规则，可以通过指定*includeFilters*和*excludeFilters* 参数的方式实现。每一个filter都需要配置type属性和expression属性，前者指定过滤的类型，后者表示相应的过滤规则。下面是可供配置的Filter一览：

***annotation***

annotation 是默认的过滤类型，表示以具有某些注解的组件作为过滤对象，配置expression时指定对应的注解类即可，比如下面这个例子：

```java
@Configuration
@ComponentScan(basePackages = "com.sample",
                includeFilters = {@ComponentScan.Filter({Controller.class, Service.class})},
                excludeFilters = {@ComponentScan.Filter(Repository.class)})
public class Config {
}
```

***assignable***

assignable表示以隶属于某些类/接口的组件作为过滤对象，配置expression时指定对应的class/interface即可，比如下面这个例子：

```java
// 这里的classes相当于指定了expression
@Configuration
@ComponentScan(basePackages = "com.sample",
                includeFilters = {@ComponentScan.Filter(type = FilterType.ASSIGNABLE_TYPE,
                                                        classes = BaseService.class)})
public class Config {
}
```

***aspectj***

aspectj表示以符合特定aspectj表达式的组件作为过滤对象，配置expression时指定对应的表达式即可，比如下面这个例子：

```java
@Configuration
@ComponentScan(basePackages = "com.sample",
                includeFilters = {@ComponentScan.Filter(type = FilterType.ASPECTJ,
                                                        pattern = "com.sample..Base*+")})
public class Config {
}
```

***regex***

regex表示以符合特定正则表达式表达式的组件作为过滤对象，配置expression时指定对应的表达式即可，比如下面这个例子：

```java
@Configuration
@ComponentScan(basePackages = "com.sample",
                includeFilters = {@ComponentScan.Filter(type = FilterType.REGEX,
                                              pattern = "com\.sample\.[^.]+\.[^.]+(Impl)"")})
public class Config {
}
```

***custom***

如果上面的几种方式都无法满足要求的话，还可以通过实现`TypeFilter` 接口的方式实现自定义的Filter。比如下面这个例子：

```java
@Configuration
@ComponentScan(basePackages = "com.sample",
                includeFilters = {@ComponentScan.Filter(type = FilterType.CUSTOM,
                                                        classes = MyTypeFilter.class)})
public class Config {
}
```

另外，Spring还支持将默认的扫描规则关闭，只需要如下配置即可：

```java
@Configuration
@ComponentScan(basePackages = "com.sample", useDefaultFilters = false)
public class Config {
}
```

***

#### 在@Component注解类中定义@Bean工厂方法

通常情况下，我们会选择在标注@Configuration的配置类中配置@Bean注解的工厂方法。其实在@Component标注的类中也可以配置@Bean注解的工厂方法，只不过两者存在一些区别，下面就通过一些实例来看一下到底存在什么区别吧。

**@Configuration注解类由CGLIB增强**

标注有@Configuration注解的配置类会由CGLIB进行增强，因此在调用@Bean注解的工厂方法时会有拦截增强的过程，而@Component注解的类则没有这种特性，具体看下面这个例子：

***这种说法有一个例外，当@Bean注解的是static方法时将不存在代理增强，这是由CGLIB的特性决定的，CGLIB是通过继承被代理类并重写其方法来实现代理增强的，而static方法无法被重写。***

@Configuration注解类

```java
@Configuration
@ComponentScan(basePackages = "com.sample")
public class Config {
}
```
@Component注解类
```java
@Component
public class Plane {
}
```
测试方法

```java
public static void main(String[] args) throws NoSuchFieldException, IllegalAccessException {

    ApplicationContext applicationContext = new AnnotationConfigApplicationContext(Config.class);
    System.out.println(applicationContext.getBean("config").getClass());
    System.out.println(applicationContext.getBean("plane").getClass());
}
```

运行结果

```
class com.sample.Config$$EnhancerBySpringCGLIB$$a15e8179
class com.sample.Plane
```

从上面的运行结果可以看到，@Configuration注解类的确经过了CGLIB代理增强。

***另外，无论是@Component还是@Configuration，其注解类的父类中定义的@Bean方法也能被Spring框架发现。***

#### 自定义@Component注解Bean的命名规则

默认情况下，当一个由@Component注解的类被装配时，注解属性value的配置内容会作为Bean的名字，而当value没有被配置时一个小写开头的简单类名会被使用。如果需要自定义命名规则，可以像下面这样指定BeanNameGenerator接口实现类。

```java
@Configuration
@ComponentScan(basePackages = "org.example", nameGenerator = MyNameGenerator.class)
public class AppConfig {
    ...
}
```

```xml
<beans>
    <context:component-scan base-package="org.example"
        name-generator="org.example.MyNameGenerator" />
</beans>
```

