# Spring AOP知识点梳理

## 基于注解的AOP配置

### 开启AspectJ支持

使用@EnableAspectJAutoProxy注解开启AspectJ支持。

```java
@Configuration
@EnableAspectJAutoProxy
public class AppConfig {

}
```

或者在xml中开启支持配置。

```xml
<aop:aspectj-autoproxy/>	
```

### 切面Aspect声明

Spring会将任何注解了@Aspect的类自动注册为切面。但需要注意的是如果基于自动扫描的话，还需要增加@Component注解。

```java
@Aspect
@Component
public class SomeAspect {

}
```

### 切点pointcut声明

#### 声明pointcut

pointcut的声明包括表达式和签名两部分，前者用来标示具体感兴趣的方法，后者则用来标示当前pointcut以便于引用。

***需要注意的是：***使用签名引用pointcut遵循Java的可见性规则，比如声明为private的pointcut只能在当前类中引用。

```java
@Pointcut("execution(* someMethod(..))")// 表达式
private void somePointcut() {}// 签名，返回值必须为void
```

pointcut表达式支持以下AspectJ中的定义：

***execution：***匹配指定方法

```java
execution(modifiers-pattern? ret-type-pattern declaring-type-pattern?name-pattern(param-pattern)
            throws-pattern?)
// 其中(param-pattern)匹配规则如下:
// ()代表无参方法 (..)代表任意数量的参数 (*)代表一个任意类型的参数 (*,String)调用一个任意类型加String类型的参数
```

***within：***匹配指定类中/指定包及子包下的所有类中定义的方法

***this：***匹配使用的proxy实现指定接口的类中定义的方法，注意不能使用通配符

***target：***匹配实现指定接口的类中定义的方法，注意不能使用通配符

***args：***匹配拥有指定类型参数的方法，可以和within搭配使用

另外还有***@within（类注解）@target（类注解） @args（参数注解）@annotation（方法注解）***来匹配注解内容

**`===`重要`===`**

其中需要特别注意的是@within和@target的区别，前者适用于定义方法的类有注解的情况，而后者适用于执行方法的类有注解的情况。这里的定义包括重载。具体可以看下面的规则：

父类没有注解，子类有注解时：@target对父类方法及重载的父类方法都有效，而@within则只对重载过的父类方法有效。

父类有注解，子类没有注解时：@target对父类方法及重载的父类方法都无效，因为执行的是子类的实例，但子类上并无注解，而@within则对父类方法有效，对重载过的父类方法无效。因为重载在这里就意味着重新定义。

***bean：***spring特有，用来匹配指定bean，支持通配符模式来指定一组bean

#### 组合pointcut

可以通过&&，||，！运算符来组合多个pointcut以达到一些特殊的配置，比如下面这个例子：

```java
@Pointcut("execution(public * *(..))") // 匹配任何公共方法
private void anyPublicOperation() {}

@Pointcut("within(com.xyz.someapp.trading..*)") // 匹配trading包下所有类中的方法
private void inTrading() {}

@Pointcut("anyPublicOperation() && inTrading()") // 匹配trading包下所有类的公共方法
private void tradingOperation() {}
```

### 通知Advice声明

#### 通知种类

***Before通知***

Before通知主要用于在指定方法处理前增加前置处理过程。

```java
@Aspect
public class BeforeExample {

    @Before("execution(* com.xyz.myapp.dao.*.*(..))")
    public void doAccessCheck() {
        // ...
    }

}
```

***After通知***

After通知在指定方法返回后调用(包括正常返回/异常返回)，可以用来定义资源释放处理。

***After throwing通知***

After throwing通知在指定方法抛出异常时调用，使用时可以指定throwing属性来获得实际排除的异常。

如果在通知方法入参指定异常类型，则当异常类型不满足条件时将不会触发方法调用。

```java
@AfterThrowing(value = "aTargetTest()", throwing = "ex")
public void afterThrowingTest(IOException ex) {
  // ...
}
```

***After returning通知***

After returning通知在方法正常返回时调用，可以通过指定returning属性来获得方法的返回值。

同样，如果指定返回值类型不符，则方法不会被调用。

```java
@AfterReturning(value = "aTargetTest()", returning = "retVal")
public void afterReturningTest(int retVal) {
    System.out.println("#####" + retVal);
}
```

***Around通知***

Around通知环绕于整个方法的执行过程，你可以决定在什么时候调用真正的方法，甚至不调用实际的方法也没有关系。尽管Around通知功能强大，但如果以上通知能满足开发需求的情况下还是选择简单的为好。

Around通知方法支持传入ProceedingJoinPoint参数来执行真正的方法，需要注意的是该参数必须为第一个参数，否则会报错。

```java
// 可以完全不调用对象方法，直接返回结果
@Around(value = "aTargetTest()")
public int aroundTest() {
    return 1000;
}
```

```java
// 也可以像下面这样覆盖方法执行的整个过程
@Around(value = "aTargetTest()")
public Object aroundTest(ProceedingJoinPoint pjp) throws Throwable{

    System.out.println("before...");

    Object retVal;
    try {

        retVal =  pjp.proceed();

    } catch (Throwable throwable) {
        System.out.println("after throwing...");
        throw throwable;
    } finally {
        System.out.println("after...");
    }

    System.out.println("after returning...");
    return retVal;
}
```

#### 通知传参

***通过JoinPoint访问方法信息***

通过在通知方法入参指定JoinPoint类型的参数，我们可以获取指定方法的方法名，参数，方法所在类等信息。

需要注意的是JoinPoint类型的参数需要在第一个，否则会报错。

***通过切点匹配类型来获取参数***

通过在通知表达式中指定诸如上面提到的args，this，target以及注解匹配类型，我们可以获取到对应的参数值。下面通过一些例子来具体说明。

使用args的例子：

```java
@Pointcut("com.xyz.myapp.SystemArchitecture.dataAccessOperation() && args(account,..)")
private void accountDataAccessOperation(Account account) {}

@Before("accountDataAccessOperation(account)")
public void validateAccount(Account account) {
    // ...
}
```
使用target/this的例子：这里可以分别获取到代理对象和被代理对象
```java
@Before(value = "aTargetTest() && target(something)")
public void beforeTest(int retVal, ISomething something) {
    // ...
}
```
```java
@Before(value = "aTargetTest() && this(something)")
public void beforeTest(int retVal, ISomething something) {
    // ...
}
```
使用注解匹配类型的例子：这里可以获取到注解属性

```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
public @interface Custom {
    String value() default "";
}
```
```java
@Before(value = "aTargetTest() && @target(custom)")
public void beforeTest(int retVal, Custom custom) {
    // ...
}
```

#### 指定参数匹配名

可以通过各个通知类型的argNames属性来配置需要匹配的参数名字，具体看下面的例子。

如果指定了argNames，那么表达式中的变量名可以和方法中的参数名不一致，只要保证变量名和argNames的配置一致即可。

如果第一个参数是JoinPoint相关类型的话，则可以不用配置在argNames中。

```java
@Before(value="com.xyz.lib.Pointcuts.anyPublicMethod() && target(bean) && @annotation(auditable)",
        argNames="bean,auditable")
public void audit(Object bean, Auditable auditable) {
    AuditCode code = auditable.value();
    // ... use code and bean
}
```

#### 指定通知方法执行的顺序

通知方法的执行顺序可以通过@Order注解来自定义，可以注解在通知方法上，也可以注解在通知方法所在的切面类上，这样Spring就会按照指定的顺序来执行各个通知方法。

#### 扩展

***Advice通知类型***

- Before advice：在join point之前执行，除非抛出异常，否则无法阻止执行流程。
- After returning advice：在join point正常结束后执行。
- After throwing advice：在异常发生时执行。
- After advice：在join point结束（正常/异常）后执行。
- Around  advice：环绕join point，不仅可以在join point前后自定义执行操作，还能选择是继续执行流程还是返回一个特殊值或是抛出一个异常来中断执行流程。

***join point***

Spring AOP目前只支持***方法执行连接点***， 如果需要使用其他join point可以考虑使用AspectJ来扩展。



## 使用Introduction

### 为现有类引入新方法

通过上面的知识梳理我们知道了可以运用切面编程来为指定类的方法添加一些共通的功能。那么如果需要给指定类增加原本不存在的方法应该怎么做呢？答案就是使用Introduction。

下面通过一个例子来说明应该怎么使用Introduction。

我们假定有一个工人接口，定义了工作方法。

```java
public interface IWorker {
    void work();
}
```

下面是工人接口的其中一个实现。

```java
@Component("defaultWorker")
public class WorkerDefaultImpl implements IWorker{
    public void work() {
        System.out.println("default working...");
    }
}
```

现在我们想要给所有的工人接口实现类加一项跳舞的技能，在不修改原先代码的前提下可以通过Introduction来实现。

首先我们需要定义一个切面类，并在其中定义Introduction。这里我们通过注解的方式来实现，使用的注解是@DeclareParents，它包含两个属性：value配置需要增加功能的对象，而defaultImpl定义了新增功能的默认实现。

```java
@DeclareParents(value = "com.sample.IWorker+", defaultImpl = DanceableDefaultImpl.class)
private static IDanceable danceable;
```

```java
public class DanceableDefaultImpl implements IDanceable{
    public void dance() {
        System.out.println("default dancing...");
    }
}
```

下面我们来看看这些工人们是不是真的能跳舞了。

```java
public static void main(String[] args) {

    //...
    IWorker worker = (IWorker) applicationContext.getBean("defaultWorker");
    worker.work();
    IDanceable dancer = (IDanceable) worker;
    dancer.dance();
    //...
}
/**
输出内容
default working...
default dancing...
**/
```

通过输出内容我们可以看到，工人们确实Get了跳舞这项新技能。

### 在现有类方法中使用指定接口方法

使用Introduction还可以实现在现有类方法中调用指定接口的方法。

下面还是通过工人的例子来看一下具体该怎么使用Introduction的这一特性。

现在我们想让工人在工作前都能先跳支舞，那应该怎么做呢？答案很简单，使用前面提到过的Before通知。

```java
@DeclareParents(value = "com.sample.IWorker+", defaultImpl = DanceableDefaultImpl.class)
private static IDanceable danceable;

// this(danceable)代表代理类实现IDanceable接口
@Before("target(com.sample.IWorker) && this(danceable)")
public void beforeWork(IDanceable danceable){
    danceable.dance();
}
```

下面我们来看看这些工人们会不会在工作前先跳支舞呢。

```java
public static void main(String[] args) {

    //...
    IWorker worker = (IWorker) applicationContext.getBean("defaultWorker");
    //...
}
/**
输出内容
default dancing...
default working...
**/
```
从输出内容可以看到，工人确实在工作前给我们跳了一支舞。