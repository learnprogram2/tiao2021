Flink 1.10 TaskManager 内存管理优化
===

### FLink 内存模型介绍:
![](https://ask.qcloudimg.com/http-save/yehe-1130324/p0r0ihbhu1.png?imageView2/2/w/1620)
![](https://ci.apache.org/projects/flink/flink-docs-release-1.11/fig/detailed-mem-model.svg)
从整体上看, FLink的 taskManager就是一个JVM, 有Heap 和 off-heap 两块

Flink 中有两个主要的内存使用者:
•用户代码中的作业 task 算子
•Flink 框架本身的内部数据结构, 网络缓冲区(Network Buffers)等

用户代码可以直接访问所有的内存类型：JVM 堆、Direct 和 Native 内存

有两种供作业 Task 使用并由 Flink 严格控制的 Off-Heap 内存: 
•Managed Memory (Off-Heap)
•网络缓冲区 (Network Buffers)




https://www.ershicimi.com/p/afa46cf80acce6422da9e511a5bec06e

博客英文地址：https://flink.apache.org/news/2020/04/21/memory-management-improvements-flink-1.10.html
作者: Andrey Zagrebin















