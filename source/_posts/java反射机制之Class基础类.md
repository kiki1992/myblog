---
title: java反射机制之Class基础类
date: 2018-01-11 15:17:32
summary: 本篇对支撑java反射机制的基础类--Class类进行了简单的源码分析整理。
tags: [java反射机制]
---

## java反射机制基础之Class类

### 概要

> * Class对象代表一个正在运行的Java应用中包含的类和接口。举例来说，枚举代表一种类而注解代表一种接口。
> * 所有元素类型和维度相同的数组共享一个Class对象。
> * Java中的基本类型和关键字void都可以用Class代表。
> * Class类没有公共的构造方法，这意味着你不能直接实例化一个Class对象，所有Class对象加载都是有Java虚拟机来完成的。
> * 通过Class我们可以获取到例如类名，类方法等各种和类相关的信息，而这也是实现Java反射机制的基础。

### 主要方法

***getClasses/getDeclaredClasses***

getClasses用于获取指定Class以及父类和祖先类中定义的所有公共类/接口。

getDeclaredClasses则用于获取当前Class中定义的public/protected/private/default类和接口。

下面是使用getClasses/getDeclaredClasses方法的例子：

```java
public class ForeFather {

    public class FF_pub{}

    private class FF_pri{}

    protected class FF_pro{}

    class FF_def{}

    public interface IFF_pub{}

    private interface IFF_pri{}

    protected interface IFF_pro{}

    interface IFF_def{}


}

public class Father extends ForeFather{

    public class F_pub{}

    private class F_pri{}

    protected class F_pro{}

    class F_def{}

    public interface IF_pub{}

    private interface IF_pri{}

    protected interface IF_pro{}

    interface IF_def{}

}

public class Me extends Father{

    public class M_pub{}

    private class M_pri{}

    protected class M_pro{}

    class M_def{}

    public interface IM_pub{}

    private interface IM_pri{}

    protected interface IM_pro{}

    interface IM_def{}

    public static void main(String[] args) {

        Class[] clzs = Me.class.getClasses();
        System.out.println("###getClasses");
        for (Class clz : clzs) {
            System.out.println(clz);
        }
        System.out.println("###getDeclaredClasses");
        Class[] dClzs = Me.class.getDeclaredClasses();
        for (Class dClz : dClzs) {
            System.out.println(dClz);
        }
    }

}
```

输出结果：

```
###getClasses
interface com.sample.cls.Me$IM_pub
class com.sample.cls.Me$M_pub
interface com.sample.cls.Father$IF_pub
class com.sample.cls.Father$F_pub
interface com.sample.cls.ForeFather$IFF_pub
class com.sample.cls.ForeFather$FF_pub
###getDeclaredClasses
interface com.sample.cls.Me$IM_def
interface com.sample.cls.Me$IM_pro
interface com.sample.cls.Me$IM_pri
interface com.sample.cls.Me$IM_pub
class com.sample.cls.Me$M_def
class com.sample.cls.Me$M_pro
class com.sample.cls.Me$M_pri
class com.sample.cls.Me$M_pub
```

***getSuperclass***

获取指定Class的父类，如果指定Class代表Object类，接口，基本类型或者void，则返回null。数组的父类则是Object。

下面是使用getSuperclass方法的例子：

```java
public static void main(String[] args) throws Exception {

    Class hashMapCls = HashMap.class;
    System.out.println("super class of HashMap is " + hashMapCls.getSuperclass());

    Class ObjectCls = Object.class;
    System.out.println("super class of Object is " + ObjectCls.getSuperclass());

    Class intCls = int.class;
    System.out.println("super class of int is " + intCls.getSuperclass());

    Class runnableCls = Runnable.class;
    System.out.println("super class of Runnable is " + runnableCls.getSuperclass());

    Class voidCls = void.class;
    System.out.println("super class of void is " + voidCls.getSuperclass());

    Class arrayClz = String[].class;
    System.out.println("super class of array is " + arrayClz.getSuperclass());

}
```

输出结果：

```
super class of HashMap is class java.util.AbstractMap
super class of Object is null
super class of int is null
super class of Runnable is null
super class of void is null
super class of array is class java.lang.Object
```

***getGenericSuperclass***

获取指定Class直接父类的Type。如果直接父类是ParameterizedType，则可以使用ParameterizedType的相关方法获取到泛型类型等信息。

下面是使用getGenericSuperclass的例子：

```java
static class Sample<T>{

    private Class clzT;

    public Sample() {

        Type superClzType = this.getClass().getGenericSuperclass();
        if (superClzType instanceof ParameterizedType) {
            clzT = (Class) ((ParameterizedType) superClzType).getActualTypeArguments()[0];
        } else {
            clzT = null;
        }

        System.out.println("Class of T is " + (clzT == null ? null : clzT.getName()));
    }

}

static class Sample1 extends Sample<String>{}

static class Sample2 extends Sample<Integer>{}

public static void main(String[] args) throws Exception {

    new Sample1();
    new Sample2();

}
```

输出结果：

```
Class of T is java.lang.String
Class of T is java.lang.Integer
```

***getInterfaces***

获取当前Class实现的接口，如果当前对象代表一个类，调用本方法将其实现的接口按照定义的顺序返回，而如果当前对象代表一个接口，方法则按照定义的顺序返回继承的接口。如果当前对象代表一个数组，则返回Cloneable和Serializable接口。

**需要注意的是：**如果不是显示定义的接口，只能通过父类调用getInterfaces获取。

下面是使用getInterfaces方法的例子：

```java
interface ISample1{}
interface ISample2{}

class SampleMix implements ISample2, ISample1{}

interface ISampleMix extends ISample1, ISample2{}

public static void main(String[] args) throws Exception {

    Class[] sampleMixInterfaces = SampleMix.class.getInterfaces();
    for (Class sampleMIxInterface : sampleMixInterfaces) {
        System.out.println(sampleMIxInterface);
    }
    System.out.println("####");
    Class[] iSampleMixInterfaces = ISampleMix.class.getInterfaces();
    for (Class iSampleMixInterface : iSampleMixInterfaces) {
        System.out.println(iSampleMixInterface);
    }
    System.out.println("####");
    Class[] arrayInterfaces = String[].class.getInterfaces();
    for (Class arrayInterface : arrayInterfaces) {
        System.out.println(arrayInterface);
    }

}
```

输出结果：

```
interface com.sample.TestMain$ISample2
interface com.sample.TestMain$ISample1
####
interface com.sample.TestMain$ISample1
interface com.sample.TestMain$ISample2
####
interface java.lang.Cloneable
interface java.io.Serializable
```

***getGenericInterfaces***

本方法和getInterfaces方法类似，区别在于getGenericInterfaces返回的是接口的Type，而Type中可能包含泛型信息。

***getComponentType***

用于获取数组元素的类型，如果当前对象不是数组，则返回null。

下面是使用getComponentType方法的例子：

```java
public static void main(String[] args) throws Exception {
    System.out.println(Integer.class.getComponentType());
    System.out.println(String[].class.getComponentType());
}
```

输出结果：

```
null
class java.lang.String
```

***getModifiers***

使用getModifiers方法可以获得包含指定类或者接口修饰符信息的int数字，需要配合java.lang.reflect.Modifier类来对解析数字包含的意义。

下面是使用getModifiers方法的例子：

```java
static class Sample1 extends Sample<String>{}

public static void main(String[] args) throws Exception {
    int mod = Sample1.class.getModifiers();
    System.out.println("is static : " + Modifier.isStatic(mod));
    System.out.println("is interface : " + Modifier.isInterface(mod));
}
```

输出结果：

```
is static : true
is interface : false
```

***getEnclosingMethod***

如果某个类是定义在方法中的内部类或匿名类，调用getEnclosingMethod方法可以知道具体是哪个方法。

下面是使用getEnclosingMethod方法的例子：

```java
static abstract class InnerClass {
    abstract void innerMethod();
}

public static void test() {

    new InnerClass() {
        @Override
        void innerMethod() {
            System.out.println(this.getClass().getEnclosingMethod());
        }
    }.innerMethod();

}

public static void test1() {

    class MethodInnerClass {
        MethodInnerClass () {
            System.out.println(this.getClass().getEnclosingMethod());
        }
    }

    new MethodInnerClass();
}

public static void main(String[] args) throws Exception {
    test();
    test1();
}
```

输出结果：

```
public static void com.sample.TestMain.test()
public static void com.sample.TestMain.test1()
```

***getEnclosingConstructor***

如果某个类是定义在构造方法中的内部类或匿名类，调用getEnclosingConstructor方法可以知道具体是哪个构造方法。

```java
static abstract class InnerClass {
    abstract void innerMethod();
}

public TestMain() {
    new InnerClass() {
        @Override
        void innerMethod() {
            System.out.println(this.getClass().getEnclosingConstructor());
        }
    }.innerMethod();
}

public static void main(String[] args) throws Exception {
    new TestMain();
}
```

输出结果：

```
public com.sample.executor.TestMain(int)
```

***getEnclosingClass/getDeclaringClass***

如果某个类定义在其他类内部，调用getEnclosingClass方法可以知道具体是哪个类（直属）。getDeclaringClass方法和getEnclosingClass类似，区别在于getDeclaringClass对匿名内部类以及方法/构造器内部定义的类不起作用。

下面是使用getEnclosingClass/getDeclaringClass方法的例子：

```java
    class SomeInnerClass{}

    public static Class otherInnerClz() {

        class OtherInnerClass{}

        return OtherInnerClass.class;
    }

    public static void main(String[] args) throws Exception {
        // 匿名内部类
        new Object() {
            public void test() {
                System.out.println(this.getClass().getDeclaringClass());
                System.out.println(this.getClass().getEnclosingClass());
            }
        }.test();
        // 方法内部类
        System.out.println(otherInnerClz().getDeclaringClass());
        System.out.println(otherInnerClz().getEnclosingClass());
        // 普通内部类
        System.out.println(SomeInnerClass.class.getDeclaringClass());
        System.out.println(SomeInnerClass.class.getEnclosingClass());
    }
```

输出结果：

```
null
class com.sample.executor.TestMain
null
class com.sample.executor.TestMain
class com.sample.executor.TestMain
class com.sample.executor.TestMain
```

***getFields***

调用本方法可以获得当前类/接口及父/祖先类/接口中定义的公共域。

