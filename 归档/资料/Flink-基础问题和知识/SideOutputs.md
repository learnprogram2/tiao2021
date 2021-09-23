### Side Outputs
除了上个操作生成的mainStream, 我们也可以在操作里生产任意数量的侧输出流. 每个侧输出流里的类型也不用和主流一样.
每个侧输出流需要定一个`OutputTag`来标识.

提交数据给侧输出流可以通过很多function(operator), 输出到Context 里面
```java
ProcessFunction
KeyedProcessFunction
CoProcessFunction
KeyedCoProcessFunction
ProcessWindowFunction
ProcessAllWindowFunction
// 使用
ctx.output(outputTag, val);
```
用的时候可以从主流里面拿: `mainDataStream.getSideOutput(outputTag);`


### Handling Application Parameters
所有的Flink应用都需要依赖外部的配置参数, 用来指定Source和sink的地址信息, 系统并行度, 运行时参数, 还有其他乱七八糟的参数.
Flink给我们提供了简单基本的参数工具`ParameterTool.clas`, 不一定要直接用这个tool, 其他框架的也适用.

#### 1. 拿到配置参数放到ParameterTool里.
tool里存储的是kv map. 可以从文件, 地址, 流里面, 从命令行里, 从系统参数里拿到配置, 用, 就像map用就好了.
```java
    ParameterTool parameter = ParameterTool.fromPropertiesFile(propertiesFile);
    ParameterTool parameter = ParameterTool.fromArgs(args);
    ParameterTool parameter = ParameterTool.fromSystemProperties();
```

#### 2. 在环境里注册参数
`env.getConfig().setGlobalJobParameters(parameters);`, 到时候也从env里拿就好了.


### Testing
Flinke提供了集成的测试工具.
1. 测试userDefinedFunction, 就正常测就好了.
2. 测试Flink Job:
    Flink提供了MiniClusterWithClientResource做本地的测试集成集群.
    使用内置集群做集成测试的建议:
    1. 为了不用重写pipline代码, 把source和sink做成可插拔的, 测试时候可以复用.
    2. 本地测试时候也要把并行度开大, 发现问题.
    3. 创建Cluster在上面标注@Class, 让所有的共享.
    4. 如果pipline里面需要操作state, 可以开启checkpointing, 然后重启试试. 重启: 在function里面抛一个错就重启了.


### Experimental Features(实验室功能)

#### Reinterpreting a pre-partitioned data stream as keyed stream
将预先分区过的stream转义成KeyedStream, 来避免shuffling. 前提是, 重新转义的流必须是按照明确的规则预先分区好的, 就像keyBy();
应用场景可以 放在两个job之间做materialized(固定) shuffle. 第一个job执行keyBy的shuffle 然后固化每个output到一个partition里, 第二个job从里面取, 每个并行度的实例去取对应的一个.
emmmm....


### ScalaAPI/JavaAPI 的扩展
为了让公共的JavaAPI和ScalaAPI一致, 一些各自更好的处理的实现从标准API里单独出来了.

Scala扩展API使用了隐式转换实现了扩展. 下面是一些api列表.
Java引入了Lambda表达式.  这里的举例就是普通的使用了一下lambda


### Project Configuration
每个Flink应用都依赖一堆的FlinkLibraries, 下面就介绍Flink应用一般都使用什么lib

#### 1. Flink Core and Application Dependencies:
用户应用一般有两个大类的依赖:
- *Flink Core Dependencies:* Flink自己的, 比如coordination, networking, checkpoints, failover, APIs, operations(比如windowing), resourceManagement...
    这些核心类都打包在了`flink-dist.jar`, 有点像JDK的rt.jar之类的.
    core了里面没有包括Connector为了避免有太多的不用的东西.
- *User Application Dependencies:* 是所有的connectors, formats或者其他的自己用的.
    这些用户应用的依赖被打包到了`application.jar`.

#### 2. Setting up a Project: Basic Dependencies
每个应用都至少需要core. `flink-streaming-java_2.11`, scope是provided/
如果不设置成provided, 那么轻则jar变大, 重则版本冲突.



Adding Connector and Library Dependencies




Scala Versions





Hadoop Dependencies





Maven Quickstart









