---
title: Spring核心技术之基于注解的容器配置
date: 2018-01-18 16:17:32
summary: 本篇对Spring中基于注解的容器配置做了简单的整理，并配以实例说明。
tags: [Spring核心技术,Spring注解]
---

### Bean自动注入 -- @Autowired VS @Resource

Spring支持两种基于注解的Bean自动注入方式，一种是框架自带的@Autowired注解方式，另一种则是基于JSR-250的@Resource注解方式。下面就来细看一下这两种注解方式。

***@Autowired 注解***

* 驱动类

  @Autowired注解由AutowiredAnnotationBeanPostProcessor类处理。

* 使用范围

  @Autowired注解可以应用于构造方法，field，set方法以及其他任意命名的方法，下面分别举例说明。

  **field/set方法**

  ```java
  public class Plane {

      @Autowired
      private Engine engine;

      private Pilot pilot;

      @Autowired
      public void setPilot(Pilot pilot) {
          this.pilot = pilot;
      }

  }
  ```

  **构造方法** Spring4.3之后的版本支持只存在单个构造器时省略@Autowired配置

  ```java
  public class Plane {

      @Autowired
      private Engine engine;

      private Pilot pilot;
      // 即使和field注解方式混用也没问题
      @Autowired
      public Plane(Pilot pilot) {
          this.pilot = pilot;
      }

  }

  ```
  **任意方法**

    ```java
    public class Plane {

        private Engine engine;

        private Pilot pilot;

        @Autowired
        public void init(Pilot pilot, Engine engine) {
            this.pilot = pilot;
            this.engine = engine;
        }

    }

    ```



* Bean匹配规则

  @Autowired注解严格按照by-Type规则匹配Bean，注意是**严格按照**，网上很多默认按照by-Type规则匹配的说法其实并不准确。具体的匹配规则如下：

  **如果没有配合@Qualifier注解使用**

  寻找符合类型条件的Bean，找到则执行注入。如果有多个Bean符合条件则会报错，这种情况可以使用@Qualifier/@Primary或者对应的xml配置来规避。具体看下面的例子。

  报错的情况：

  ```java
  public class Plane {

      @Autowired
      private Engine engine;

      @Autowired
      private Pilot pilot;

  }
  ```

  ```xml
  <bean id="engine" class="com.jpeverything.springsrc.Model.Engine">
      <constructor-arg name="name" value="default"></constructor-arg>
  </bean>

  <bean id="pilotDefault" class="com.jpeverything.springsrc.Model.Pilot">
      <property name="name" value="default"></property>
  </bean>

  <bean id="pilotSuper" class="com.jpeverything.springsrc.Model.Pilot">
      <property name="name" value="super"></property>
  </bean>
  ```

  ```
  Caused by: org.springframework.beans.factory.NoUniqueBeanDefinitionException: No qualifying bean of type [com.jpeverything.springsrc.Model.Pilot] is defined: expected single matching bean but found 2: pilotDefault,pilotSuper
  ```

  使用@Primary注解/对应的xml配置，可以这样改：

  @Primary注解/对应的xml配置的作用是将指定bean标记为首要的，这样当存在多个满足条件的Bean时，标记的Bean会被注入。

  ```xml
  <bean id="engine" class="com.jpeverything.springsrc.Model.Engine">
      <constructor-arg name="name" value="default"></constructor-arg>
  </bean>

  <bean id="pilotDefault" class="com.jpeverything.springsrc.Model.Pilot">
      <property name="name" value="default"></property>
  </bean>

  <bean id="pilotSuper" class="com.jpeverything.springsrc.Model.Pilot" primary="true">
      <property name="name" value="super"></property>
  </bean>
  ```

  **如果配合@Qualifier注解使用**

  寻找和@Qualifier注解配置一致的Bean(前提是类型一致)，因为Bean没有配置Qualifier属性时默认使用的是id作为Qualifier，就造成了by-Name匹配的假象，实际还是必须首先满足类型一致的条件。

  看一下下面这个例子：

  ```java
  @Component
  public class Plane {
      
      @Autowired
      @Qualifier("engine")
      private Engine engine;

      @Autowired
      private Pilot pilot;
      

  }

  ```

  ```xml
  <bean id="engine" class="com.jpeverything.springsrc.Model.Good">
    <property name="name" value="default"></property>
    <qualifier value="engine"></qualifier>
  </bean>
  ```

  ```
  No qualifying bean of type [com.jpeverything.springsrc.Model.Engine] found for dependency [com.jpeverything.springsrc.Model.Engine]: expected at least 1 bean which qualifies as autowire candidate for this dependency. Dependency annotations: {@org.springframework.beans.factory.annotation.Autowired(required=true)}
  ```

  上面的例子中我们配置了一个qualifier为engine却完全是另外一种不相干的类型的bean，如果以by-Name的规则来匹配的话，报错信息应该是类型不匹配，但我们看到上面提示的是找不到对应类型的Bean，这也说明即使配置了@Qualifier注解，@Autowired还是按照严格按照by-Type来匹配的。

  **non-required配置**

  由@Autowired注解的Bean可以设置注入是否必须，如果需要兼容不存在可注入Bean的情况，可以通过以下三种方式配置：

  1.设置@Autowired的required为false

  2.结合Java的Optional使用。需要JDK1.8+

  3.使用@Nullable注解。需要Spring5.0+

  ```xml
  <!-- javax.annotation.Nullable依赖 -->
  <dependency>
      <groupId>com.google.code.findbugs</groupId>
      <artifactId>jsr305</artifactId>
      <version>3.0.2</version>
  </dependency>
  ```



***@Resource注解***

* 驱动类

  @Resource注解由CommonAnnotationBeanPostProcessor类处理。


* 使用范围

  Spring框架支持基于field和set方法的@Resource注解，比如你可以像下面这样使用@Resource注解。

  ```java
  public class Plane {

      @Resource(name="engine")
      private Engine engine;

      private Pilot pilot;

      @Resource(name="pilot")
      public void setPilot(Pilot pilot) {
          this.pilot = pilot;
      }

  }

  ```

* Bean匹配规则

  @Resource注解默认按照by-Name的规则来寻找相应的Bean，同时会在一定条件下转为by-Type规则匹配。具体规则如下：

  **如果配置了name属性:**

  严格按照name配置匹配相应的Bean。所以下面的例子中自动注入将会失败。Spring会试图寻找名为pilot的Bean，而这里配置了pilot1。

  ```java
  public class Plane {

      @Resource(name="engine")
      private Engine engine;

      private Pilot pilot;

      @Resource(name="pilot")
      public void setPilot(Pilot pilot) {
          this.pilot = pilot;
      }

  }
  ```

  ```xml
  <bean id="engine" class="com.jpeverything.springsrc.Model.Engine">
      <constructor-arg name="name" value="default"></constructor-arg>
  </bean>

  <bean id="pilot1" class="com.jpeverything.springsrc.Model.Pilot">
      <property name="name" value="default"></property>
  </bean>
  ```

   **如果没有配置name属性:**

  首先使用默认的name属性（field名）寻找相应的Bean，如果通过name找不到，再通过类型匹配的方式。因此下面的例子将会匹配成功。
  ```java
  public class Plane {

      @Resource(name="engine")
      private Engine engine;

      private Pilot pilot;

      @Resource
      public void setPilot(Pilot pilot) {
          this.pilot = pilot;
      }

  }
  ```
  ```xml
  <bean id="engine" class="com.jpeverything.springsrc.Model.Engine">
      <constructor-arg name="name" value="default"></constructor-arg>
  </bean>

  <bean id="pilot1" class="com.jpeverything.springsrc.Model.Pilot">
      <property name="name" value="default"></property>
  </bean>
  ```

  另外，如果配置了集合或者数组类型，那么所有符合类型的Bean都将被注入。比如把上面的例子改成这样，结果就是所有Pilot Bean都会被注入。
  ```java
  public class Plane {

      @Resource(name="engine")
      private Engine engine;

      private List<Pilot> pilots;

      @Resource
      public void setPilot(List<Pilot> pilots) {
          this.pilots = pilots;
      }

  }
  ```

