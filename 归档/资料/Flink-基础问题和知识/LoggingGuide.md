## Logging 指南
> 大概的就介绍了logging的三个元件, TODO 但是我不太明白他说的第三方框架(log4j) 和 抽象层(SLF4J))的区别.

### Java Logging Basics
本章讲的是Java logging基础, 包括怎么创建logs, 流行的logging框架, 怎么创建最好的logLayouts并且用Appenders把log发送到不同的目的地, 还有线程context和markers.

### Logging Frameworks
java里面的logging需要依赖logging框架, 这些框架提供了objects, 方法和创建发送log的必要配置. 
Java原生的框架在`java.util.logging`包里面. 也有第三方的框架(Log4j, Logback, tinylog). 我们还可以使用抽象层(SLF4J, ApacheCommonsLogging), 这会把我们的代码从基础气质框架里分离开, 我们就能随意换logging框架了.
对大部分人来说Log4j是不错的, 良好的性能, 高度配置. 如果我们使用Java框架集成, 最好把SLF4J和Log4j结合起来用, 实现最大兼容.

### Abstraction Layers
SLF4J之类的抽象层, 解耦了我们的程序和基础日志框架. 允许我们根据需要更改logging框架. 这个抽象层提供了通用API, 并且根据applicationClasspath里面可用的框架来决定用哪个日志框架.
如果一个框架在classpath里面找不到, 抽象层会禁用log调用. 抽象层在我们计划升级或更改日志框架的时候非常有用.

### Java Logging Components
Java提供了自定义和可扩展的logging系统. 除了Java提供的基础logging, 我们可以轻松的使用其他的loggign替代方案. 不同的解决方案提供了不同的方法API, 但都有相同的基本结构:
Logging API有三个核心构建组成:
1. Logger: 负责捕捉events, 把他们传递给合适的Appender.
2. Appenders: 也叫handlers, 负责把logEvent发到目的地, 使用Layout来格式化.
3. Layouts: 也叫formatter, 负责格式化logEvent的数据格式.
整个的过程是, logger把log记录在LogRecord里面, 交给Appender, Appender使用Layout进行格式化, 然后发送到console/file/.. 里. 除此之外, 还可以在Appenders里面指定Filters.

### Configuration
大多数情况下, Logging框架通过配置文件来配置. 配置文件放在application里, 运行时候logging框架拿出来用. 虽然我们可以用代码配置, 但是, 可以用代码配置, 但配置文件比较好因为可以把所有的配置在同一个地方. 

#### 1. `java.util.logging`
配置文件放在logging.properties里面.
```properties
# Appender 的配置
java.util.logging.FileHandler.pattern = %h/java%u.log
java.util.logging.FileHandler.limit = 50000
java.util.logging.FileHandler.count = 1
java.util.logging.FileHandler.formatter = java.util.logging.XmlFormatter
```
#### 2. Log4j
支持不同格式的配置文件, 找不到就默认console输出. 
#### 3. Logback
logback.xml文件. 也可以用`logback.groovy`文件配置. 

### Loggers
logger是出发logEvents的Obj, 在我们的java代码里创建使用, 一个class里面可以有多个独立的logger触发不同的logEvent. 也可以继承嵌套.

#### 1. 创建logger
不同的框架大概都相同, 传递一个名字来拿到一个logger, 如果同名的logger有了, 就把那个传给你.
```java
Logger logger = Logger.getLogger(MyClass.class.getName());
```
#### 2. Logging Events
日志级别可用在过滤logEvnet到不同的Appender里.  

### Appenders
appender通过layout格式化之后选择发送到哪, 可以发送到多个地方. 

#### 1. adding appenders
不同的框架提供功能相似但实现不同的appenders. 
java自带的logging, 我们可以调用logger.addHandler添加appender到logger里面. 
一般都通过配置文件添加,
```txt
handlers=java.util.logging.ConsoleHandler, java.util.logging.FileHandler
```
#### 2. appenders' types
**ConsoleAppender**
```xml
<!--Log4j2框架-->
<?xml version="1.0" encoding="UTF-8"?>
 <Configuration status="warn" name="MyApp">
   <Appenders>
	 <!--绑定system.out-->
     <Console name="MyAppender" target="SYSTEM_OUT">
       <PatternLayout pattern="%m%n"/>
     </Console>
   </Appenders>
   <Loggers>
     <!--root级别的日志, 所有级别的-->
     <Root level="error">
       <AppenderRef ref="MyAppender"/>
     </Root>
   </Loggers>
 </Configuration>

<!--Logback框架-->
<configuration>
  <appender name="MyAppender" class="ch.qos.Logback.core.ConsoleAppender">
    <encoder>
      <pattern>%m%n</pattern>
    </encoder>
  </appender>
  <root level="error">
    <appender-ref ref="MyAppender" />
  </root>
</configuration>
```
**FileAppenders**
**SyslogAppender**
把logEntries发送到syslog服务器里. syslog专门收集不同服务的log. 
**还有一些其他的.**

#### 3. 选择一个appender
取决于我们的需求. 如果我们想确认会不会影响我们的性能, 可以看[性能对比](https://www.loggly.com/blog/benchmarking-java-logging-frameworks/)

### Layouts
logging框架提供了很多layout, 有plaintTxt, HEML, syslog, XML, JSON... (Java原生的JUL把layouts叫Formatter)
JUL只有simpleLayouts和XMLLayouts两个. 

#### 1. configuring a Layout
Layouts一般在配置文件里配置, 从java7开始, `SimpleFormatters`可以用system参数配置. 
log4j和logback最常用PatternLayout. 可以很好的格式化我们的logEvent.
```xml
<!--log4j里面的配置, 配置在appender里面-->
<PatternLayout pattern="%d{HH:mm:ss.SSS} [%t] %-5level %logger{36} - %msg%n"/>
```
#### 2. Changing Layouts
JUL里面可以修改appender的layout在代码里. 我们可以创建一个新的handler然后自己设置formatter. 
**Using Custom Layouts**
JUL里可以自定义Layouts.

### Log Levels
JUL里面的level有七个级别, 还有一个ALL和OFF, 让logger去log所有级别的log或者关闭. 
```xml
SEVERE(HIGHEST LEVEL)
WARNING
INFO
CONFIG
FINE
FINER
FINEST(LOWEST LEVEL)
```
#### 1. Setting a Log Level
配置文件理赔就好了.

#### 2. Logging Stack Traces
在log4j或者logback里面的PatternLayout里面加上%xEx这种格式可以输出stackTraces. 
```txt
 [%p] %t: %m<b>%xEx</b>
```
其他的框架配置.....







