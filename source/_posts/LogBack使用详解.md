---
title: Logback使用详解（更新中）
date: 2017-12-18 11:11:32
summary: 本篇围绕Logback的配置展开对其使用方法的详细说明。
tags: 日志框架
---

## Logback简介
Logback是由大名鼎鼎的日志系统Log4j创始人Ceki Gülcü主导开发的又一开源日志系统。作为Log4j的后继项目，Logback在很多方面都优于现存其他日志系统。除此之外，它还提供了许多其他日志系统中没有且实用的特性。

使用Logback时可以配合SLF4J日志门面。作为一个用于日志系统的简单Facade，SLF4J可以服务于各个日志系统，因此你可以在需要的时候方便地替换实际使用的日志系统。值得一提的是，SLF4J同样也是由Ceki Gülcü主导开发的。

要开始使用Logback，只需要引入以下jar包依赖：
*logback-core.jar*   *logback-classic.jar* -- 本文发布时的jar包版本为1.1.11
如果需要配合SLF4J使用，那么还需要引入以下jar包依赖：
*slf4j-api.jar*  -- 本文发布时的jar包版本为1.7.25

## 基本配置
Logback可以基于xml或者groovy配置，这里只记录xml的配置方法。首先来看一个配置的例子：

```xml
<configuration>
  <appender name="STDOUT" class="ch.qos.logback.core.ConsoleAppender">
    <encoder>
      <pattern>%d{HH:mm:ss.SSS} [%thread] %-5level %logger{36} - %msg%n</pattern>
    </encoder>
  </appender>

  <root level="debug">
    <appender-ref ref="STDOUT" />
  </root>
</configuration>
```

### configuration节点配置

xml配置的根节点为configuration，可以自定义debug和scan属性。

configuration节点可以由0到多个appender节点，0到多个logger节点以及一个root节点构成。

#### debug属性

配置debug属性为true时，Logback会输出记录其内部状态的日志，前提是配置文件存在且格式正确。
```xml
<configuration debug="true">
```

#### scan属性

配置scan属性为true可以让Logback在配置文件发生变化时自动重载，它有一个配套的属性scanPeriod来定义扫描的间隔，可以配置的单位有[milliseconds, seconds, minutes,hours]，默认使用milliseconds，建议配置时带上单位。

```xml
<configuration scan="true" scanPeriod="10 seconds">
```
***需要注意的是，***上面配置的扫描间隔并不代表配置文件重载扫描真正执行的时间间隔，只有当执行了N次日志输出操作并且过了扫描间隔之后才会触发配置文件的重载扫描操作。而这里的N也不是一个固定值，其取值取决于当前应用输出日志的频率。N的一个参考值是默认值16，在一些CPU密集型的应用中这个值可以达到2^16 (= 65536)。

### Logger节点配置
Logger节点可以配置name,level以及additivity属性，除了name属性以外其他两个属性都是可选配置。
***name属性: *** 定义logger的标示，一般使用包名或者类名。
***level属性: *** 可以指定TRACE, DEBUG, INFO, WARN, ERROR, ALL或 OFF，以及特殊值INHERITED/NULL(代表强制继承父logger的level配置)。如果省略level属性就会继承父logger的配置。
关于logger的level属性继承可以看下面这个例子：
![logger-inherit](/myblog/images/Logback/logger-inherit.png)
可以看到在名为a.b.c的logger在没有指定level属性的情况下继承了父logger-a.b的level设置。而图中的ROOT表示root节点，它是所有logger的祖先。因此在父logger及其他祖先logger都没有配置level属性时，当前logger会继承ROOT的配置。
***additivity属性: *** 指定logger是否继承父logger输出源appender，如果指定为false，则子logger只会在自己定义的appender中输出日志，默认为true。
Logger节点可以包含多个appender-ref子节点来指定日志的输出源
***appender-ref节点: ***通过节点的ref属性配置指定的输出源appender
**配置例**
```xml
<configuration>
...
<logger name="com.demo" level="DEBUG" additivity="false">
    <appender-ref ref="STDOUT" />
    <appender-ref ref="FILEOUT" />    
</logger>
</configuration>
```

### root节点配置
root节点其实就是一个特殊的logger节点，只不过它只有一个必须属性level。至于为什么没有logger的name和additivity属性，原因很简单，root logger的名字就是ROOT无法更改自然就没有name属性，而root logger又是所有logger的祖先所有也就不存在继承appender的属性additivity了。
另外，和其他logger一样，root logger同样支持指定多个输出源appender。
**配置例**
```xml
<configuration>
...
<root level="DEBUG">
    <appender-ref ref="STDOUT" />
    <appender-ref ref="FILEOUT" />    
</root>
</configuration>
```

### 变量替换配置
Logback支持变量的动态替换，变量替换使用${}表达式。用于替换的变量值可以来自当前文件，Logback的上下文，JVM运行时参数以及环境变量。
#### 变量配置
**xml配置**
xml配置中可以使用property或者variable节点配置单个变量或者从外部文件/资源中获取多个变量配置。
如果是直接定义一个变量可以如下配置：
```xml
...
<property name="logpath" value="/home/demo/app.log" />
<appender name="FILE" class="ch.qos.logback.core.FileAppender">
  <file>${logpath}</file>
  <encoder>
    <pattern>%msg%n</pattern>
  </encoder>
</appender>
...
```
如果需要从外部文件/资源获取多个变量可以如下配置：
```xml
...
<!-- 从文件系统读取 -->
...
<property file="/home/demo/config/logback.properties" />
<!-- 从classpath读取 -->
<property resource="logback.properties" /> 
...
```
**java系统变量配置**
如果需要通过java系统变量配置，可以直接在命令行做如下指定：
```shell
java -Dlogpath="/home/demo/app.log" MyApp
```

**通过JNDI获取**
动态变量值还可以通过JNDI获取，默认获取的变量有效范围为local，具体配置如下：
```xml
...
<insertFromJNDI env-entry-name="java:comp/env/appName" as="appName" scope="system" />
...
```

#### 变量值替换
**普通替换**
变量值替换使用${}标签，配置例如下：
```xml
...
<logger name="com.demo" level="${demoLoggerLevel}">
</logger>
...
```
**默认值设置**
变量值的替换支持默认值设置，配置例如下：
```xml
...
<!-- 默认logger等级debug -->
<logger name="com.demo" level="${demoLoggerLevel:-DEBUG}">
</logger>
...
```
**变量嵌套**
变量值的替换还支持嵌套配置，即可以在${}中嵌套其他${}表达式。配置例如下：
```xml
...
<!-- 默认值动态配置 -->
<logger name="com.demo" level="${demoLoggerLevel:${defaultLevel}}">
</logger>
<!-- 变量名配置 -->
<logger name="${${demoLogger}.name}" level="${${demoLogger}.level}">
</logger>
...
```