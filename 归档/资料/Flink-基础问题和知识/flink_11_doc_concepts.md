# Concepts

## Overview
上面的手动训练解释了Flink的状态流的基本概念, 也提供了怎么使用这些机制. 状态流处理在上面的`DataPiplines&ETL`里面介绍过概念, 也在`FaultTolerance`里面进一步使用了. 实时流也在`StreamingAnalytics`里面介绍了.

Concepts这一节, 介绍更深的理解Flink架构怎么运行的

### Flink's APIs
Flink提供了多层的抽象结构为我们的开发: ![](https://ci.apache.org/projects/flink/flink-docs-release-1.11/fig/levels_of_abstraction.svg)

1. 最底层提供了 `Stateful and Timely Stream processing`, 状态流是通过ProcessFunction内置到DataStreamAPI里面的. 这让我们可以自由的处理n个流的事件, 并且提供了持久化, 容错的state. 除此之外, 用户可以注册eventTime和processingTime的callback, 允许我们可以实现更复杂的计算.
2. 实际上, 很多应用不需要使用底层api, 可以使用CoreAPI进行编程就好了.DataStream/DataSetAPIs.这些流API提供了通用的data处理模块, 比如多种形式的用户自定义转换: Joins, aggregations, windows, state... API里面计算用的DataType是我们编程语言里面的class.
	底层的ProcessFunction和DataStreamAPI世纪城的, 可以根据需要使用底层的抽象(ProcessFunction). DataSetAPI提供了其他的有界数据的原语: 迭代/循环...
3. TableAPI是声明表的DSL. 可以动态的修改表(表示stream的时候), tableAPI遵循可扩展的关系模型: 有schema(有点像DB), 提供其他的对比操作(select, project, join, group-by..) tableAPI不提供具体操作代码, 表达能力比CorAPI差, 但是更简单.
	我们可以在table和dataStream/dataSet之间无缝转换, 混合使用.
4. 最高层的抽象是SQL, 有点像tableAPI, 但是直接使用SQL查询. SQL抽象与TableAPI紧密结合, 可以在tableAPI上面的表里查询.



## Stateful Stream Processing
### State是什么
stream里面的操作有的是每个event独立的, 有的需要记住跨event的信息(比如window), 这些操作叫做有状态的.
状态操作举例:
1. app在寻找具体event模式的时候, state可以存储event出现的顺序. 
2. 在整合一段时间的数据时候, state应该拿着在pending的汇总.
3. 在训练机器学习模型的时候, state应该拿着当前的模型参数.
Flink需要了解state, 使用checkpoint和savepoint做容错处理. state也允许rescal, 说明flink管理state跨并行度之间的重分配. 
`Queryable state`允许我们可以在flink外面拿到state.
在使用state的时候, 也可以阅读`Flink state backends`, flink 提供了不同的statebackend来指定怎么存储. 

### Keyed State
keyed-state可以看做成内置的k-v存储, state被分区和分布紧随着stream, 可以被statefulOperator读到. 此外, k-v state只能在keyedStream里面. 对齐key和state可以保证state的更新都是本地操作, 确保了没有事务开销的一致性. key的对齐也允许Flink把state分发, 透明的调整stream的分区. 
![keyedStateConceple](https://ci.apache.org/projects/flink/flink-docs-release-1.11/fig/state_partitioning.svg)
keyed-state进一步被组织成keyGroup, keyGroup是原子单位, Flink可以使用它重新分发state.keyGroup的数量就是并行度的数量, 在运行时候, 每一个keyedOperator的并行度示例都可以使用一个或者多个keyGroup.

### State持久化
Flink通过`stream replay`和`checkpointing`的结合来实现容错处理. checkpoint会在每一个inputStream标记一个特殊点和对应的operator的对应state. stream流可以从checkpoint里面恢复, checkpoint维持着一致性, 通过恢复operator的state并且从checkpoint开始replay数据(exactly-once语义). 
checkpoint的间隔是过分容错和恢复时间的平衡. 
容错机制不断地做dataflow的snapshot, 如果是state比较小, 那么非常快. state被存在配置的地方(一般是nfs之类的分布文件系统).

遇到了程序出错的时候, flink把流停住. 系统然后重启operators, 把他们设置到最新的checkpoint状态. inputStream也恢复到state快照的状态, 所有的record在重启之后的dataflow运行, 并且保证不会影响之前的checkpoint state.
> checkpoint默认关闭. 为了保证这种guarantee, dataStream的source必须保证可以把流恢复到之前的一个点上. 
> 因为flink的checkpoint是通过分布式的snapshot实现的, 我们可以混用这两个词. 一般来说snapshot代表checkpoint或者savepoint.

#### 1. Checkpoint
flink的容错机制最重要的就是画分布的stream和operatorState的一致性快照. 这些快照在failure的时候就当成一致性的checkpoint. 制作这些snapshot的机制叫做`[Lightweight Asynchronous Snapshots for Distributed Dataflows](http://arxiv.org/abs/1506.08603)`. 灵感来自于分布式快照的标准的Chandy-Lamport算法, 并且特殊的定制了.
要记住, checkpoint所有相关的操作都可以异步, checkpoint屏障没有锁的步骤, operator也可以异步的快照state.
flink1.11里面checkpoint可以不需要对齐做snapshot. 之前需要

##### 1.1 Barriers(屏障)
flink分布快照最重要的一个element就是流的barrier. 这些屏障在流的里面, 随着record流动, 是六的一部分. barrier不会跳过record, 它们严格的排队, 把record分割成一个一个的set. 每个barrier带着snapshot的ID, 前面的record就都到snapshot进去了.  barrier不会打扰flow 所以非常的轻便. 多个barrier代表着不同的snapshot可以同一时间在流里面. 不同的snapshot也可以并发的进行. 
![checkpoint和它的barrier](https://ci.apache.org/projects/flink/flink-docs-release-1.11/fig/stream_barriers.svg)
stream的屏障在每一个并行度的流里. 每一个并行流里面的屏障前面最后一条会报告给checkpoint的coordinator(就是jobManager)
屏障会接着跟着流流动, 中间的operator接收到了所有inputStream的屏障, 就发给下游一个屏障, 最后sink收到了所有的屏障, 就会告诉checkpointCoordinator自己的snapshot完成了, 所有的sink都完成了之后就结束了. 
一旦snapshot_n完成了, job不会再向stream要Sn之前的数据了, 因为都已经完整的消费过了.
![snapshot](https://ci.apache.org/projects/flink/flink-docs-release-1.11/fig/stream_aligning.svg)
接收多个inputStream的时候, 需要对齐barrier. 上面的图也介绍了. 
- 在接入的六里面一拿到barrier_n, 就不接收这个流里的数据了, 等把其他的流里面的barrier都接受到. 不然的话怕混了.
- 收到了所有的barrier_n, 就开始发送缓存的record, 接着再把这个barrier_n提交出去.
- 开始做state的快照, 然后接着从inputStream里面接收数据(先拿缓存里面的) 
- operator会把state异步的写道stateBackend里面
所有接收多个流的operator都需要对齐, 或者接受流的shuffle之后的数据也需要. 

##### 1.2 快照 operator state
在operator包含有任何形式的state的时候, 这些state必须做snapshot.
在收到了所有的snapshot_barrier的时候就可以开始快照了, 在提交这些barrier之前完成. 这个时间带你上, 所有的更新都做了, 还没有之后record的影响. 因为state可能很大, 存在配置的_statebackend_里. 默认的存放在jobmanager的内存里, 但在生产上要用分布式文件系统可靠一点. state存好了之后, operator知道了checkpoint, 然后把barrier提交给下游, 自己接着处理.
snapshot包含:
	- 每一个source并行度的offset/position(snapshot开始的时候)
	- 每一个operator的state存放的地址.
	![checkpoint_process](https://ci.apache.org/projects/flink/flink-docs-release-1.11/fig/checkpointing.svg)
	
##### 1.3 恢复 recovery
恢复机制比较直接, 遇到了錯誤, flink选择最新完成的checkpoint. 系统会重放整个dataflow, 然后给每个operator恢复state. source设置到当时的position. 
如果state是增长式的snapshot的时候, operator从上一次最近的fullsnapshot开始, 然后把之后的snapshot修改放到这个state上面.
可以看[RestartStrategies](https://ci.apache.org/projects/flink/flink-docs-release-1.11/dev/task_failure_recovery.html#restart-strategies)

#### 2. 不对齐的checkpointing
Flink1.11开始, checkpoint也可以不对齐了. 基本思想是 checkpoint通过包含了in-flight的data, 来克服这些数据带来的扰乱. 会把这个barrier里面的record和state一起快照.
注意: 这个方法更接近`Chandy-Lamport`算法, 但是Flink还会insert barrier在source, 来避免checkpointCoordinator的放太多~
![how handle the unaligned checkpoint](https://ci.apache.org/projects/flink/flink-docs-release-1.11/fig/stream_unaligning.svg)
上面, unaligned的checkpoint分三步:
	- operator堆缓存里面的第一个barrier做checkpoint
	- 快速的把这个barrier放在输出buffer的最后, 为了发送做准备. 
	- 把所有的barrier之前应该处理的数据做一步存储, 然后创建快照. 
不对齐的checkpoint确保了barrier尽快的到达sink. 最合适那种有一个比较慢的流动的应用, 对齐时间可能长达几个小时. 但是它增加了I/O压力, 其他的限制可以看[ops](https://ci.apache.org/projects/flink/flink-docs-release-1.11/ops/state/checkpoints.html#unaligned-checkpoints)
注意: savepoint是必须对齐的.

###### 不对齐的recovery
operator收起恢复in-flight数据. 除此之外和对齐的recovery一样.

#### 3. state backends`
k-v的额外数据存放在state backend里面. 一个state backend存在内存里的hashMap里, 另一个stateBackend使用RockesDB做kv的存储. 
除了定义state的structure之外, stateBackend也实现了存储checkpoint的state. 
![](https://ci.apache.org/projects/flink/flink-docs-release-1.11/fig/checkpoints.svg)

#### 4. Savepoints
所有使用checkpoints的程序都可以从savepoint里面恢复, savepoint允许我们不损失任何state的时候来升级程序.
savepoint是手动触发的checkpoint, 会堆整个程序做快照, 并且把它写到stateBackend. 依赖于定期的checkpoint.
savepoint类似于checkpoints, 只是这是手动触发的, 不会自动失效.

#### Exactly Once vs AtLeastOnce
对齐的步骤可能会给stream处理添加latency. 一般情况下这额外的latency是几毫秒, 但是有的时候增长很多. 对于要求很低latency的程序, flink有一个开关在stream checkpoint对齐的时候跳过.checkpoint快照在operator收到每个input的barrier的时候还会接着做. 
对齐跳过之后, operator接着处理input, 甚至在checkpoint_n到了之后有跳过了很多个barrier. 在恢复的是哦胡, record会重复的出现, 因为record可能在checkpoint_n里面处理过了, 然后还会在n之后重放. 
就是过去了, 然后不记录它为成功, restore的时候肯定又来了一遍. 
注意: 对齐旨在operator有多个前置操作(join)的时候或者有多个sender(比如repartition/shuffle)的时候进行. 所以, dataflow在流里面的parallel操作的时候实际只会exactlyOnce.

### State and Fault Tolerance in Batch Programs 批处理里面的state和容错.
flink把批处理当作特殊的流处理执行, 批处理是有界的. dataSet内部就是一个流. 上面的概念也试用于批处理. 只有几个例外:
1. 不会做checkpoint, recovery会replay stream. 
2. stateful操作在dataSetAPI里面指挥用简单的in-memory/out-ofcore 数据结构, 而不是key/value的indexes.
3. dataSet API引入了特殊的同步的遍历. [iterations](https://ci.apache.org/projects/flink/flink-docs-release-1.11/dev/batch/iterations.html)




## Timely Stream Processing 实时流处理


### Introduction
实时流处理是状态流处理的扩展, 其中time在计算里起到了一定的作用. 在其它事情之外, 我们可以做时间序列的分析, 在做一定的时间段内的聚合的时候, 或者在做时间很重要的事件处理的时候. 

下面的部分会highlight一些我们在做实时Flink应用的时候应该考虑的东西. 

### 时间概念: EventTime 和 ProcessingTime
在流处理里面涉及到的时间, 一般有两个时间概念:
- Processing time: ProcessingTime指的是正在工作的机器的系统时间. 
在流处理程序跑在processing时间的时候, 所有的实践操作(like time windows)会使用各自的系统时间. 一个小时的timeWindow会包含特定operator里面处理的一个小时内的所有数据. 比如一个app在9:15开始跑, 第一个1小时window会包含从9:15到10:00的所有event, 之后的是10-11, 11-12这个时间段的.
processingTime是最简单的时间概念, 不需要在stream和机器之间做协调处理. 它提供了最好的性能和最低的latency. 然而, 在分布式和异步环境里, processingTime不代表什么意义, 因为它收到系统里面的record消费速度, 和其他的影响. 
- Event time: eventtime是每个独立的事件在发生的时候的时间. 这个时间一般在进入flink之前就已经植入到record里面了, eventTimestamp可以从每一个record里面拿到. 在eventTime里, 时间的处理取决于每个event, 不缺与其他的. eventTime程序必须指定怎么去生成`EventTimeWatermarks`. 下面会讲watermark机制.
在完美的假设下, eventTime处理会得到完全一致和确定的结果. 不管event什么时候到, 怎么排序. 但是, 除非event排序到达, eventTime处理在等待迟到的event的时候会增加latency.因为程序只能等一定的时间, 所以这也限制了eventTimeApplication的确定性.
假设所有的数据都倒了, eventTime操作会按照期待的工作, 产生一致和正确的结果.
注意, 有的时候eventTime程序处理的实施的数据, 它们会使用一些processingTime操作来保证可以很快的执行.
![two notions of time](https://ci.apache.org/projects/flink/flink-docs-release-1.11/fig/event_processing_time.svg)

### EventTime 和 Watermarks
注意: Flink实现了DataflowModel的一些技术. eventTime和watermark的介绍可以看一下两篇文章: `Streaming 101`, `DATa flow Modle paper`
支持eventTime的流式处理需要保证eventTime的进度, 比如一个hourly的window需要在eventTime过了一个小时之后被告知, 然后operator才能关闭window.
eventTime可以独立于processintTime处理, 比如,在程序里eventTime一般会落后于processingTime, 处理的速度相同. 另一方面, 有一些流处理程序只需要花费几秒钟的计算就可以处理一周的数据, 

Flink衡量eventTime的进度的机制就是watermark. watermarks随着流流动, 并且带着一个时间戳. 一个watermark(t)就表示流里面的eventTime已经到了t, 也意味着t和t之前的element都消耗完了. 
下图表示一个带有时间戳的流, watermark在流里面. 这个例子里面事件是按照时间排序的, watermark是简单的周期标记. 
![streamInOrder](https://ci.apache.org/projects/flink/flink-docs-release-1.11/fig/stream_watermark_in_order.svg)
watermark是无序流的关键, 下面展示的,event没有按照时间顺序. 一般来说watermark是stream里面的一个点. 在watermark到了一个operator的时候, 这个operatro可以把自己内部的`eventTimeClock`设置为watermark的时间.
![streamoutoforder](https://ci.apache.org/projects/flink/flink-docs-release-1.11/fig/stream_watermark_out_of_order.svg)

#### Watermarks in Parallel Streams 并发流里面的watermark
watermark在source里面或者之后创建. 每个并行度的子流里面的sourceFunction会单独的创建自己的watermark. 这些watermark定义了特定source的平行度里面. 
watermark流过workflow, 它们会推进到的没有给operator, 在operator被推进里自己的event时间之后, 他会创建一个新的watermark往下推. 
一些operator需要消费多个inputStream, 可能是union或者keyby,或者partition操作. 这样的operator的当前的eventTime是输入流里面最小的watermark里面的时间. 
下面的图里面展示了并行流里面的watermark和operator的eventTime
![EventTimeAtTheParallel](https://ci.apache.org/projects/flink/flink-docs-release-1.11/fig/parallel_streams_watermarks.svg)

### Lateness
有可能有一些element会违反watermark, 在watermark之后出现.事实上很多element可能迟到, 让指定一个时间所有的element都到的是不可能的. 还有, 即使lateness可以提高, 也不希望watermark迟到太多, 会导致eventWIndow延迟. 
这样,流式处理的程序可能会特殊的要求一些迟到的element. 迟到的element是在系统的watermrk之后出现的. [allowedLateness](https://ci.apache.org/projects/flink/flink-docs-release-1.11/dev/stream/operators/windows.html#allowed-lateness)

### Windowing
event的聚合在stream里面和批处理里面不一样, 比如在流里面不能统计所有的element. 流里面的聚合一般是window scope之内的, 比如过去五分钟之内的统计, 一百个element的聚合.
Window可以是按照时间驱动的, 也可以按照数据驱动的(100个). 一般有滚动窗口, 滑动窗口, session窗口.
![difference window](https://ci.apache.org/projects/flink/flink-docs-release-1.11/fig/windows.svg)
这个[blog](https://flink.apache.org/news/2015/12/04/Introducing-windows.html)里面介绍了一些window例子.



## Flink 架构
flink是分布式系统, 需要高效的分配和管理计算资源, 来执行流式应用. flink集成在通用的集群资源管理上, 比如YARN, Mesos或者Kubernets, 也可以作为单独的集群运行.
这一部分包含了Flink的架构, 描述了flink主要的component和程序的交互, 还有故障恢复.

### Anatomy(剖析) of a Flink Cluster
flink运行时候包含两个processes, JobManager还有多个TaskManager
![Runtime](https://ci.apache.org/projects/flink/flink-docs-release-1.11/fig/processes.svg)
client不是runtime和程序执行的一部分, 他只用来把dataflow发到jobManager. 执行命令用的.
JobManager和taskManager可以不同的方式启动, 直接搭建集群, 或者用container, 或者使用YARN和Mesos管理资源. TaskManager连接到JobManager, 宣布自己是可用的, 然后接受工作.

#### JobManager 
jobmanager负责所有的东西, 关于coordinate分布式执行的东西. 它决定了什么时候执行下一个task, task结束/失败之后的响应, 平衡checkpoint, 恢复. 涉及到三个component.
- ResourceManager: 负责resource的分配, 管理taskSlot这个基本的资源单位. 
	flink实现了多个ResourceManager来适应不同的环境(YARN, Mesos, K8s...) 在standalone的启动, resourceManager只能使用已有的slots, 不能创建新的TaskManager.
- Dispatcher(调度员)
	调度员提供了restful接口, 来接受FlinkApplication执行, 为每个提交的job都开启新的jobMaster. 还提供flink的WebUI.
- JobMaster: 负责管理一个[jobGraph](https://ci.apache.org/projects/flink/flink-docs-release-1.11/concepts/glossary.html#logical-graph)的执行. 多个job可以在FlinkCluster里面同时执行, 每个都有自己的jobMaster.
JobManager[里面也有HA, 一个是leader, 其他的都standby.](https://ci.apache.org/projects/flink/flink-docs-release-1.11/ops/jobmanager_high_availability.html)

#### TaskManagers
也叫workers, 负责执行dataflow里面的任务, 缓存和交换dataStream.
至少有一个TaskManager, taskManager里面最小的资源分配是一个slot. taskSlot的数量代表着可以并行的task数量, 多个operator可以组成operatorChains在一个taskSlot里面运行.

### Task and Operator Chains
为了分布式执行, flink把operator连起来组成tasks. 每个task可以在一个线程里面执行. 把operator连起来组成task是非常有效的优化: 减少了线程切换的成本和缓存成本, 提升了整体的吞吐量降低了latency. 可以配置链接的细节.
下图里面简单的dataFlow可以用5个subtask执行, 有五个并行的线程.
![demo](https://ci.apache.org/projects/flink/flink-docs-release-1.11/fig/tasks_chains.svg)

### Task Slots 和 resources
每个Taskmanager都是一个JVM程序, 在多个县城里执行一个或多个子任务. TaskManager接受task的数量叫做taskSlot.
每个 task slot 代表着taskmanager里面一个固定的子资源. slot意味着每个subtask不会和其它job的子任务争夺资源, 有一定的保留内存. 没有CPU隔离, slot只会分割task的内存.
通过调整taskSlot的数量,我们可以定义subTask之间怎么区分. 每个Task Manager如果只有一个slot就意味着每个task都在不同的JVM里面运行. 多个slot共享同一个JVM. 在同一个JVM里面的task共享TCP链接, 还有心跳messages. 也共享dataSets还有dataStuctures, 来较少每个task的压力.
![twoTaskManager](https://ci.apache.org/projects/flink/flink-docs-release-1.11/fig/tasks_slots.svg)
默认的, FLink允许不同的task之间共享同一个slot, 只要他们是同一个job下面的. 这可能让一个slot里面放着整个job的pipeline. 这个slotSharing有两个优点:
- flink cluster需要slot和job的最高并行度一样, 不需要计算job里面有多少个task.
- 有更好的资源利用率. 如果没有slotSharing, 非密集型的source/map之类的任务将会把后面subtask都阻塞. 如果有了slotSharing, 可以提升并行度, 下面就提升2到六. 可以充分利用slot, 也确保了heavy的subtask可以在taskManager之间公平的分配. 
![slorSharing](https://ci.apache.org/projects/flink/flink-docs-release-1.11/fig/slot_sharing.svg)

### Flink Application Execution
Flink应用就是用户程序里面的main方法里面产生的FlinkJobs, job可以在本地JVM或者分布多台机器里面执行. `ExecutionEnvironment`提供了控制job执行的方法, 和外界的交互. 
Flink的job可以提交到长时间运行的FlinkSessionCluster里面, 或者专用的FlinkJObCluster里, 或者FLink集群里面.
上面几个选项的差异主要是集群的生命周期还有资源隔离保证不同. 

#### Flink session cluster(Flink Cluster in session mode)
- cluster lifecycle: 在FlinkSessionCluster里面, client连接到预先存在的, 长期运行的集群. 左右的job都做完了之后, cluster还会接着运行直到手动的结束session. 
- Resource Isolation: taskManager的slot被ResourceManager在job提交的时候分配, 然后再job结束的时候释放. 所有的job都共享cluster, 集群资源存在竞争, 比如网络带宽. 共享的局限性在于一个taskManager上面一个subtask失败之后会影响到taskManager上面运行的所有的job失败.
- 其他的要考虑的: 预先存在的cluster可以节省资源分配和启动taskManager的时间.

#### Flink Job Cluster(Flink Cluster in job mode)
- cluster lifecycle: 在job集群里面, 集群管理器(YARN/K8s)回味每一个job启动建立一个自己的集群. client收钱从clusterManager请求资源来启动jobManagaer, 然后把job提交到Dispatcher.
	TaskManager基于资源的分配懒加载, job完成之后, cluster就被拆掉了. 
- resource 隔离: jobManger的error只会影响到自己的job.
- 其他要考虑的: 因为ResourceManager需要取执行还有等待额外的resourceManagement component来启动taskManager来处理分配资源, FlinkJObManager更适合单独长期运行的大Job, 有更高的稳定性要求, 对启动时间不太要求.

#### Flink Application Cluster
- Cluster lifecycle: 应用集群是专用的flink集群, 只会从main()方法里面执行, 而不是从client拿到job. job提交只需要一步, 不需要开启FlinkCluster然后提交. 我们把application逻辑和依赖打包成一个可执行的jar, 然后cluster入口(ApplicationClusterEntryPoint)负责调用main方法, 来吧jobGraph拿出来. 这就像我们开发其他的k8s程序一样. 生命周期就是这个job.
- 资源隔离: 一个flink应用集群, resourceManager和Dispatcher是一个flink应用的scope, 比FlinkSessionCluster更关注分界点.

























