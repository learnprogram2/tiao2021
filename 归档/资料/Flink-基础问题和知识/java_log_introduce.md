> I read this artical from Zhihu. a little know about the relationships between Abstraction Layers and LoggingFrameworks


1. Java中的IO在最初提供了System.err.println, 只能输出到控制台,让控制台一团糟.

2. log4j诞生了
![Log4j 体系图](https://pic2.zhimg.com/80/v2-e28a982dd94430c5254c7ec97a0a5821_720w.jpg)

3. Log4j 在Apache开源以后, 作者又写了一个工具，叫做logback, 速度更快. 


4. 用户仅使用抽象层, 实现底层可以切换
作者看到各种日志实现除了java.util.logging, log4j 之外, 还有logback,tinylog 等其他工具.
提供抽象层(Simple Logging Facade for Java, 简称SLF4J):
	1. 用户用这个抽象层的API来写日志, 底层具体用什么日志工具不用关心, 方便移植. 
	2. 对于Log4j,JDKlogging,tinylog等工具, 需要一个适配层, 把SLF4J的API转化成具体工具的调用接口

5. 结果:
Logback这个工具也是出自作者, 直接实现了SLF4J的API, 不需要适配层, 效率最高, SLFJ4+Logback组合受欢迎, 大有超越Apache Common Logging + Log4j 之势

