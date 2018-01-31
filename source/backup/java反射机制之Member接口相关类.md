---
title: java反射机制之Member
date: 2018-01-18 15:17:32
summary: 本篇对支撑java反射机制的基础--Member接口及相关实现类进行了简单的源码分析整理。
tags: [java反射机制]
---

## Member接口

```java
public interface Member 
```

### 概要

> * Member接口代表了单个成员（域变量或方法）或构造器的辨识信息。
> * Member分为PUBLIC（0）和DECLARED（1）两种，其中PUBLIC辨识代表一个类/接口中所有公共成员（包括从父类/祖先类继承的），而DECLARED辨识则代表本类/接口中所有成员（包括公共的，私有的等等）。

### 主要方法

***getDeclaringClass***

```java
public Class<?> getDeclaringClass();
```

getDeclaringClass方法返回定义当前Member的类对象。

***getName***

```java
public String getName();
```

getName方法返回当前Member的名字（不包含类型，修饰符信息）。

***getModifiers***

```java
public int getModifiers();
```
getModifiers方法返回当前Member的修饰符信息。

***isSynthetic***

```java
public boolean isSynthetic();
```

isSynthetic方法用于判断当前Member是否由编译器引入。



***



## Field类

```java
public final
class Field extends AccessibleObject implements Member
```

### 概要

> * Field类提供了类/接口的单个域信息，以及动态访问域的方法。包括类域和实例域。
> * Field允许调用get/set方法时的向上转型，而当有向下转型时则会抛出IllegalArgumentException。

### 主要方法

***isEnumConstant***

用于判断当前域是否为枚举类对象。

***getType***

```java
public Class<?> getType()
```

用于获取当前域的类型（不包含泛型信息），比如java.lang.String。

***getGenericType***

```java
public Type getGenericType() 
```

用于获取当前域的类型（包含泛型信息），比如java.util.List`<`java.lang.String`>`。

***get***

```java
public Object get(Object obj)
    throws IllegalArgumentException, IllegalAccessException
```

用于获取指定域的设值。

如果指定域为类域，入参obj实际并无意义，可以直接传null。

如果指定域为私有域，需要先调用setAccessible方法设置访问权限。

如果指定域为基本类型，通过get方法返回的值会被自动包装。比如获取int类型域的值将返回Integer类型的对象。

另外Field类还提供了用于获取基本类型值的方法。例如getBoolean，getChar，getByte等等。

***set***

```java
public void set(Object obj, Object value)
    throws IllegalArgumentException, IllegalAccessException
```

用于设置指定域的设值。

如果指定域为类域，入参obj实际并无意义，可以直接传null。

如果指定域为基本类型，value会被自动解包。

final类型的域也可以通过本方法设值，前提是调用setAccessible方法成功且当前域不是类域，不过需要注意的是，设值final类型的域只有在对象反序列化或者重新构造时才有意义，设值时final域尚为空，对程序的其他部分也尚未可访问。除此以外的使用可能带来不可预期的结果。

另外Field类还提供了用于设置基本类型值的方法。例如setBoolean，setChar，setByte等等。

***AnnotatedType***

用于获得代表注解对象field的AnnotatedType。可以从返回的AnnotatedType获得field的类型。

关于AnnotatedType可以参考以下链接：

https://stackoverflow.com/questions/20364236/java-8-class-to-annotatedtype



***



## Executable类

```java
public abstract class Executable extends AccessibleObject
    implements Member, GenericDeclaration 
```

### 概要

> * Executable类是另外两个重要的Member类--Method和Constructor的共同父类。

### 主要方法

***getAnnotatedExceptionTypes***

```java
public AnnotatedType[] getAnnotatedExceptionTypes()
```

用于获取代表方法/构造器定义的异常类型的AnnotatedType数组。

比如你可以像下面这样定义一个表示异常等级的注解，然后通过getAnnotatedExceptionTypes获取注解信息。

需要注意的是注解必须标注有TYPE_USE，否则无法获取。

```java
@Target({TYPE_USE})
@Retention(RUNTIME)
public @interface ExLevel {
    int level() default 0;
}
```

```java
public void someMethod() throws @ExLevel IOException {

}

public static void main(String[] args) throws NoSuchMethodException {

    Method someMethod = Me.class.getMethod("someMethod", null);
    AnnotatedType[] exAnnotatedTypes = someMethod.getAnnotatedExceptionTypes();
    // Annotation @com.sample.cls.ExLevel(level=0) on class java.io.IOException
    System.out.println("Annotation " + exAnnotatedType.getAnnotations()[0] + " on " + exAnnotatedTypes[0].getType());

}
```

***getAnnotatedParameterTypes***

```java
public AnnotatedType[] getAnnotatedParameterTypes() 
```

用于获取代表方法/构造器定义的参数类型的AnnotatedType数组。

***getAnnotatedReceiverType***

```java
public AnnotatedType getAnnotatedReceiverType()
```

用于获取代表方法/构造器定义的Receiver参数类型的AnnotatedType。

Receiver参数指的是实例方法定义的this入参，或内部类构造方法中定义的.this入参。

```java
public void someMethod(@Custom Me this)  {}

public static void someStaticMethod() {}

public static void main(String[] args) throws NoSuchMethodException {

    Method someMethod = Me.class.getMethod("someMethod", null);
    AnnotatedType receiverType = someMethod.getAnnotatedReceiverType();
    // receiverType of instance method class com.sample.cls.Me with 1 annotation(s)
    System.out.println("receiverType of instance method "
            + (receiverType != null ? receiverType.getType() + " with " + receiverType.getAnnotations().length + " annotation(s)" : "not exists"));

    Method someStaticMethod = Me.class.getMethod("someStaticMethod", null);
    AnnotatedType receiverTypeStatic = someStaticMethod.getAnnotatedReceiverType();
    // receiverType of static method not exists
    System.out.println("receiverType of static method "
            + (receiverTypeStatic != null ? receiverTypeStatic.getType() : "not exists"));

}
```

***getAnnotatedReturnType***

```java
public abstract AnnotatedType getAnnotatedReturnType();
```

用于获取代表方法/构造器定义的返回值类型的AnnotatedType。

***getDeclaredAnnotations***

用于获取方法/构造器上定义的注解。

```java
public Annotation[] getDeclaredAnnotations()
```

***getAnnotationsByType***

```java
public <T extends Annotation> T[] getAnnotationsByType(Class<T> annotationClass)
```

和getDeclaredAnnotations方法的区别是可以指定获取的注解类型。并且可以获得可重复注解信息。

```java
@MyResource(value = "aa")
@MyResource(value = "bb")
public void someMethodOvd(){}

public static void main(String[] args) throws NoSuchMethodException {

    Annotation[] annotations = Me.class.getDeclaredMethod("someMethodOvd", null).getDeclaredAnnotationsByType(MyResource.class);
    for (Annotation annotation : annotations) {
        System.out.println(annotation);
    }

}
```

***getAnnotation***

```java
public <T extends Annotation> T getAnnotation(Class<T> annotationClass)
```

和getAnnotationsByType方法的区别是当可重复注解配置多个时无法获取注解信息。

***getParameterAnnotations***

```java
public abstract Annotation[][] getParameterAnnotations()
```

用于获取方法/构造器参数注解一览。

***isSynthetic***

```java
public boolean isSynthetic()
```

用于判断构造器是否由编译器创建。

***isVarArgs***

```java
public boolean isVarArgs()
```

用于判断是否支持可变参数。

***getGenericExceptionTypes***

```java
public Type[] getGenericExceptionTypes()
```

用于获取当前方法/构造器定义的异常Type，包含泛型信息。

***getExceptionTypes***

```java
public abstract Class<?>[] getExceptionTypes()
```

和getGenericExceptionTypes方法的区别在于本方法返回值不包含泛型信息。

***getParameters***

```java
public Parameter[] getParameters() 
```

用于获取方法/构造器定义的参数一览。包含隐式传递的参数，比如内部构造器方法隐式传递了外部类实例this。关于Parameter将单独做记录整理。

***getGenericParameterTypes***

```java
public Type[] getGenericParameterTypes()
```

用于获取包含泛型信息的参数类型一览。包含隐式传递的参数，比如内部构造器方法隐式传递了外部类实例this。

***getParameterTypes***

```java
public abstract Class<?>[] getParameterTypes()
```

用于获取不包含泛型信息的参数类型一览。包含隐式传递的参数，比如内部构造器方法隐式传递了外部类实例this。

***getParameterCount***

```java
public int getParameterCount()
```

用于获取形参的个数。包含隐式传递的参数，比如内部构造器方法隐式传递了外部类实例this。

***getTypeParameters***

```java
public abstract TypeVariable<?>[] getTypeParameters()
```

用于获取方法定义的泛型变量一览。例如下面的X,Y,Z

```java
public <X, Y, Z> void someMethod()  {
}
```

***getModifiers***

```java
public abstract int getModifiers();
```

用于获取修饰符信息。

***getName***

```java
public abstract String getName()
```

用于获取方法/构造器名。

***getDeclaringClass***

```java
public abstract Class<?> getDeclaringClass()
```

用于获取定义方法/构造器的类。

## Method类







## 相关扩展内容

### ElementType

ElementType枚举类定义了注解的使用范围，具体取值如下：

***TYPE*** 适用于类，接口(包括注解)，枚举。

***FIELD*** 适用于域变量，包括枚举值。

***METHOD*** 适用于方法。

***PARAMETER*** 适用于形参。

***CONSTRUCTOR*** 适用于构造器。

***LOCAL_VARIABLE*** 适用于本地变量。

***ANNOTATION_TYPE*** 适用于注解类型。

***PACKAGE*** 适用于包。

***TYPE_PARAMETER*** 适用于类/接口的泛型参数。

***TYPE_USE(使用Type的地方)*** 包含TYPE，TYPE_PARAMETER，以及以下十二种场景：

**用于声明**

* 定义class时使用extends和implements

  ```java
  public class Son extends @SomeAnnotation Father
  ```

* 定义interface时使用extends
  ```java
  public interface ISon extends @SomeAnnotation IFather
  ```

* 定义方法返回值（包括注解类中定义的成员方法）
  ```java
  public @SomeAnnotation void someMethod() {}
  ```

* 定义方法/构造器的throws异常
  ```java
  public void someMethod() @SomeAnnotation throws Exception {}
  ```

* 类，接口，方法，构造器声明类型变量时使用extends

  ```java
  public <T extends @SomeAnnotation Comparable<? super T>> T someMethod() {
      ...
  }
  ```

* 定义类/接口域变量，以及枚举的枚举值

* 定义方法/构造器/lambda表达式的形参

* 定义方法的receiver参数 

* 定义本地变量

* 定义异常参数(try-catch)

  ```java
  try {
      ...
  } catch (@SomeAnnotation Exception e) {
      ...
  }
  ```


**用于表达式**

* 构造器调用，实例化，方法调用时的参数类型

  ```java
  Me me = new Me( new @SomeAnnotation Money());
  ```

* 通过new实例化时需要实例化的class类或者其父类
  ```java
  Me me = new @SomeAnnotation Me( new @SomeAnnotation Money());
  Inner inner = new Me.@Custom Inner();
  ```

* 初始化数组时的数据类型

  ```java
  Integer[] integers = new @SomeAnnotation Integer[10];
  ```

* 类型强转表达式声明的类型

  ```java
  Me me = (@SomeAnnotation Me) human;
  ```

* instance of后指定的类型

  ```java
  if (superman instanceof @SomeAnnotation Human) {
      ... 
  }
  ```

* 方法引用表达式指定的类型

  ```java
  Function<String,String> instanceFunc = @SomeAnnotation String::trim;
  ```