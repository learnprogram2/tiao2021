## DataTypes & Serialization

### Overview
Flink 处理dataType和序列化比较特殊, 包括flink里面的typeDescriptor, genericTypeExtraction和typeSerializationFramework. 本文描述了这个概念还有原理(rationale)

#### 1. 支持的数据类型
flink在stream/set里面的数据类型做了限制, 因为为了系统的分析数据类型来探测最有效地执行策略.
```java
// 可以有七种数据类型
Java Tuples and Scala Case Classes
Java POJOs
Primitive Types
Regular Classes
Values
Hadoop Writables
Special Types
```
1. Tuples and case classes:
... 不多说了.

2. POJOs:
需要满足一些规则的类就是POJO: public的class, 无参public构造器, getter/setter, field必须有支持的serializer.

3. 基本(Primitive)数据类型
Integer, String, Double 之类的.

4. General Class Types
遵循Java Beans规范的类基本都可以. 不满足POJO的就都当成generalClass处理了. generalType的序列化时使用的[Kryo](https://github.com/EsotericSoftware/kryo)

5. Values
value类型手动的制定serialization, 通过实现`org.apache.flinktypes.Value` 而不是指定一个序列化框架.  自己实现read,write. 标准的序列化效率低的时候就可以使用Value type了.
`CopyableValue`这个接口支持手动的指定内部克隆的逻辑.
flink准备了一些Value types对应基本数据类型(ByteValue, ShortValue, ...). 这些Value可以充当基本类型的替代品, 重用, 减少内存消耗.

6. Hadoop Writables.
可以实现`org.apache.hadoop.Writable`接口, 实现write的序列化逻辑. 

7. 特殊类型
也可以用特殊类型, 比如Scala的`Either,Option,Try`等, Java也有自己的Either的实现, 它表示一个value可能有两种类型, `left/right`. `Wither`可以在异常处理或者输出两种类型的时候使用.

8. 类型擦除(Type Erasure) & 类型推断(Type Inference)
java编译器在编译之后会扔掉很多通用的类型信息(泛型). 这个在Java里叫泛型擦除. 在runtime, 泛型不知道是那个类型.
Flink在准备program来执行的时候需要类型信息, FlinkJavaAPI尝试用不同的方式把扔掉的typeInfo重新构建然后把他存在dataset和operator里. 通过dataStream的`getType()`方法可以拿到`TypeInformation`, 里面存着Flink自己的方式表示的type.
类型推断有局限性, 需要依赖程序员的协作, 比如那些从collections创建的dataSet(env.fromCollection), 我们需要传递type信息的参数. 其他的MapFunction也需要额外的类型信息.
`ResultTypeQueryable`这个接口也已用来在传入的时候告诉API袭击返回类型. function的`input types`一般依赖于之前的operation的resultType.

#### 2. Type Handling in Flink
Flink尝试从分布式计算时候交换和存储的dataType里面推断出很多信息. 可以把它想成一个数据库存了很多table的schema. 大多谁情况flink自己结合了必要信息. 有了type信息flink可以做:
1. 使用POJO类型的时候可以按照字段进行group/joining/aggregating, typeInfo可以让Flink提前检查避免运行时拿不到.
2. Flin知道越多的type信息, 就可以更好地序列化和了解scheme. 在Flink内存使用非常重要. 
3. 让用户不用注册序列化框架.
一般来说, dataType一般在一些方法前调用需要: execute(), print(), count(), collect().

#### 3. Most Frequent Issues
1. 注册 subtypes: 如果方法签名只是supertype, 但用的都是subclass, 那么让flink注册一下subtype可能会让flink更快. 注册的话可以调用`env.registerType(clazz)`.

2. 注册自定义的serializer: flink依赖Kryo处理未知的types, 可以调用`env.getConfig().addDefaultKryoSerializer(clazz,serializer)`来自定义, 下一小节是介绍自定义序列化的.

3. 添加type hints: 有的时候Flink没法推断出类型, 必须传给flink`type hint`, 在Java里面需要. 
```java
DataSet<SomeType> result = dataSet
    .map(new MyGenericNonInferrableFunction<Long, SomeType>())
        .returns(SomeType.class);
```
4. 手动创建`TypeInformation`: 在Flink不能推断类型擦除后的type的时候需要我们定义. 
```java
TypeInformation<Tuple2<String, Double>> info = TypeInformation.of(new TypeHint<Tuple2<String, Double>>(){});
```

#### 4. Flink’s TypeInformation class
`TypeInformation`是所有type descriptor的基本类, 它揭示了type的基本特性, 能创建serializer, type的比较. (flink里面的comparator不知定义了顺序, 而且是key处理工具.)
Flink自己做了一些type的区分
1. 基本类型, 包装类
2. 基本类型的array和object的数组.
3. 复合类型: Tuples, POJOs, Row.
4. 辅助类型: Option, Either, Lists...
5. 泛型: 能被Kryo序列化的. (不是POJO的)
**Create TypeInformation/TypeSerializer**
`TypeInformation<String> info = TypeInformation.of(String.class);`
拿到了TypeInformation之后就调用它的`typeInfo.createSerializer(config)`方法来创建TYpeSerializer. 
config是`ExecutionConfig`, 它hold需要注册的customSerializer. 我们在stream里面可以用`getExecutionCOnfig()`拿到它. 


#### 5. ScalaAPI里面的typeInformation
...
#### 6. JavaAPI里面的TypeInformation:
泛型的时候, java会把类型信息擦除, Flink努力如还原类型信息, 通过反射. 对于返回类型取决于输入类型的时候, 还包括简单的类型推断. 
```java
    public Tuple2<T, Long> map(T value) {
        return new Tuple2<T, Long>(value, 1L);
    }
	// 有的时候Flink不能推断出泛型的信息, 需要制定Hints
	DataSet<SomeType> result = dataSet
    .map(new MyGenericNonInferrableFunction<Long, SomeType>())
        .returns(SomeType.class);
```
**Serialization of POJO types**
`PojoTypeInfo`创建所有POJO里面字段的serializer. 
如果Kryo都无法反序列化, 那么我们可以调用`evn.getConfig().enableForceAvro()`来用avro序列化POJO.
可以全部用Kryo序列化`env.getConfig().enableForceKryo();`, 然后遇到Kryo解决不了的, 添加自定义的:`env.getConfig().addDefaultKryoSerializer(Class<?> type, Class<? extends Serializer<?>> serializerClass)`

#### 7. 关闭Kryo的fallback
如果我们不想用kryo作为推断不出泛型的类型的时候的选择, 我们要制定好所有的Flink自带的不能序列的, 然后调用`env.getConfig().disableGenericTypes();`关闭Kryo的接入.

#### 8. 使用Factory定义TypeInformation
`type information factory`允许我们插件式的定义typeInformation. 我们要去实现`org.apache.flink.api.common.typeinfo.TypeInfoFactory`来返回自定义的TypeInformation.
这个factory在type提取解析的时候如果返回的类型标注了`@TypeInfo`注解的时候调用
```java
@TypeInfo(MyTupleTypeInfoFactory.class)
public class MyTuple<T0, T1> {
  public T0 myfield0;
  public T1 myfield1;
}
public class MyTupleTypeInfoFactory extends TypeInfoFactory<MyTuple> {
  @Override
  public TypeInformation<MyTuple> createTypeInfo(Type t, Map<String, TypeInformation<?>> genericParameters) {
    return new MyTupleTypeInfo(genericParameters.get("T0"), genericParameters.get("T1"));
  }
}
```


### Register custom serializer for your Flink program
Flink在不能序列化的时候就会用Kryo序列, 我们可以在Kryo里面注册自己的序列器. 我们只需要在ExecutionConfig里面注册我们的序列器就好了.
```java
env.getConfig().registerTypeWithKryoSerializer(MyCustomType.class, MyCustomSerializer.class);

env.getConfig().registerTypeWithKryoSerializer(MyCustomType.class, mySerializer);
```
注意, 我们自定义的serializer必须集成Kryo的Serializerclass. 可以看一下ProtobufSerializer和TBaseSerializer.

制定好了之后, 如果使用谷歌和apache还需要添加一些必要的protobuf-java和libthrift依赖. 


#### Issue with using Kryo’s JavaSerializer
因为Kryo可能会用错加载器, 所以, 可能会有`ClassNotFoundException`, 这种情况下我们就应该用`org.apache.flink.api.java.typeutils.runtime.kryo.JavaSerializer`, 可以再实现它确保使用用户自己代码的类加载器. 



## Managing Execution 执行管理

### Execution Configuration 执行参数配置
`StreamExecutionEnvironment`包含了ExecutionConfig, 里面允许我们取设置一些job的参数. 可以改变job默认的配置.[ConfigurationList](https://ci.apache.org/projects/flink/flink-docs-release-1.11/ops/config.html)

### ProgramPackaging and Distributed Execution, 项目打包和分布式运行
program一般达成jar包, 在cluster里面运行. 使用[CLI](https://ci.apache.org/projects/flink/flink-docs-release-1.11/ops/cli.html#command-line-interface)接口取运行jar包. 

#### 1. packaging programs:
为了支持CLI命令接口, 必须用`StreamExecutionEnvironment.getExecutionEnvironment()`这个方法拿到env. 如果jar包使用commandlinke运行的, 拿到的就是cluster的env. 其他方式拿到的是local的.
打包把main-class制定好就可以运行了.

#### 2. 总结
启动一个jar的步骤:
1. jar的mainclass制定好. 在META-INF里面文件里的`main-class`和`program-class`, programClass优先级高. 
2. 然后调用mainClass.

### Parallel Execution 并行执行
本小节讲怎么去配置并行执行的job. 
如果想使用`savepoints`, 我应该设置最大的并行度, 因为在恢复的时候, 也可以修改指定operator或整个的并行度. 这是因为Flink内部把state分配成`key-group`的partition, 有太多的key-group会损害性能. 

#### 1. 设置parallelism
可以设置不同level的并行度. operator, env的.
**operator level**
**Execution Environment Level:** env会为所有的operator指定一个默认的并行度.
**Client Level:** 在client submit job到flink的时候可以设定. 
**System Level:** 在Flink及群里可以设定所有job的默认并行度.

#### 2. Setting the maximum parallelism
默认的最大并行度大概是 `operatorParallelism + (operatorParallelism/2)`, 上下限为[128, 32768]


### Execution Plans(执行计划)
Flin自动优化并选择一个执行计划 根据不同的参数(datasize/cluster). 我们可以知道Flink会怎么执行我们的程序.

#### 1. plan visualization tool
我们从env里面拿到我们的执行计划: `env.getExecutionPlan()`, 然后flink提供了一个[可视化工具](https://flink.apache.org/visualizer/). 

#### 2. Web Interface
Flink提供了用于job submit和执行的接口, 是JobManager的监控接口. 我们可以指定程序参数在job执行前. 


### Task Failure Recovery
task报错的时候, Flink就重启相关的task来recover.
重启策略和failover策略来控制 failed&affected task的重启. Restart策略决定一个failed/affected task是否需要重启和什么时候重启. Failover策略来决定哪个task应该restart整个job.

#### 1. Restart Strategies:
没指定就用默认的. 默认的restartStrategy通过flink配置文件设置`flink-conf.yaml`. 里面的`restart-strategy`参数制定了策略. 
下面就是一些配置参数了.

#### 2. Failover Strategies
也是通过`flink-conf.yaml`文件配置. 可以配置restart all(重启整个job)和restart pipelined region.
**restart pipelined region:**
这个策略把task分成不相交的regions. 在有一个task failure了, 策略会计算region最小set, 然后重启.
一个region是pipeline里面有数据交流的task. 
1. All data exchanges in a DataStream job or Streaming Table/SQL job are pipelined.
2. All data exchanges in a Batch Table/SQL job are batched by default.
data exchange可以铜鼓ExecutionConfig里面的ExecutionMode拿到.
包含failureTask的region要重启, 如果一个region重启, 他的上游下游都要重启. 


## API Migration Guides
...  除妖影响的是用户自定义的state的TypeSerializer.旧的configSnapshot废弃了. 
























