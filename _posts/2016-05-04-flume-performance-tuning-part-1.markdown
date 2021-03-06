---
layout: post
title: "[译]Apache Flume 性能调优 (第一部分)"
date:  2016-05-04 11:40:56
categories: Hadoop
---

Apache Flume，是一个分布式的、可靠的、高可用的服务，用于收集、聚合、传输大量的事件型数据的应用组件。本篇文章是关于Apache Flume性能调优系列文章的第一部分。

在这篇文章中，我们将要讨论Flume调优时的两个基本概念：通道（Channel）和事务Batch Size(transaction batch Size)。

---



**原文链接：http://blog.cloudera.com/blog/2013/01/how-to-do-apache-flume-performance-tuning-part-1/**

**作者 : Mike Percy**

**译者：勾满誉 (me@myg0u.com)**

----

本文中的中文术语与英文术语对照：

Event :  事件

Channel : 通道

----

#### 构建一个数据流


假设你希望将一个海量的流式的用户行为日志从你的应用服务器中提取出来并存储到Hadoop进行分析。这些日志来自于你已有的应用服务器集群，并且构建一个将大量节点的数据汇总到少数几个节点的fan-in 架构 。

如果你将这些用户事件每次发送一条，而且每次都需要等待这些事件送达，你的吞吐量很可能会非常受限于网络延迟。自然，你希望将多个事件打包为一个事务，这样事务确认的开销就会摊薄到批量事务中的每一个事件，这样可以大大的提高你的吞吐量。


---

#### 通道（Channels）



下面，想象一下如果存储层突然停止工作，比如存储层网络分区离线的时候会发生什么？如果Flume agent挂掉， 那些事件会怎么样？ 即使发生了这些故障，我们也希望通过某种方法继续在应用层继续服务我们的用户，并且在服务恢复时可以恢复这些数据。为了达到这个目的，我们需要为agent建立一个缓冲机制来预防下游服务可能出现的缓慢和宕机。 在Flume中，通道就是事件在数据流中的跳板。 下面的图向你说明了通道在Flume Agent架构中的位置。


![image](/images/flume1.jpg)

----


#### 内存通道与文件通道的对比

当设计Flume系统时，一个重要的选择就是使用什么类型的通道(Channel)。在本文中，主要向你推荐两种通道，即文件通道（File Channel） 与 内存通道（Memory Channel）。 文件通道是一个持久化的通道，它将所有的事件都持久化到硬盘中。所以，即使JVM被杀掉，操作系统重启或宕机，没有被传输到下一个数据处理流程的事件将在flume agent重启时被恢复。内存通道是一个易失通道，它仅在内存中存储事件，如果JVM被杀掉或者操作系统宕机，所有储存在内存中的数据都会丢失，当然，与文件通道相比，内存通道也提供了更低的读写延迟，即使事件的Batch Size只有1。 由于在内存通道的容量受限于内存， 所以如果下游服务发生临时失败，它缓存的能力也非常有限。 而文件通道，由于硬盘空间一般比内存大得多且廉价得多，所以它可以缓存事件的能力也要比内存通道强得多。

-----

#### Flume  事件的批量处理


在上文中提到，Flume可以批量处理事件。 Batch Size是Sink或者客户端单次事务可以从通道中取得事件的最大值。调低Batch Size参数，会造成吞吐量的降低，但同时也降低了失败时，发生事件多重记录的风险。 调高Batch Size参数，可以显著提高吞吐量，也会增加延迟，但是如果发生宕机，发生事件重复记录的风险也会相应的变高。


事务（Transaction）在Flume中是一个重要概念，因为只有在后面的每一个事务都成功的时候，可靠存储与送达才有保证。举个例子来说，当数据源接收到或者生产了一条事件， 为了将事件存储到通道，首先它必须先打开通道。在事务中， 数据源将Batch Size数量的事件送入通道中，然后才确认已经提交了事务。 Sink端需要经过同样的过程，才能从通道中提取事件。


Batch Size 是由Sink配置的。Batch Size越大， Channel也就越快。 文件通道的速度差异主要体现在它必须将缓冲区完全同步到硬盘时，事务才算成功提交。 硬盘同步本身就是一个非常耗时的操作（百毫秒级别），因为它要保证数据的可靠性。同样内存通道中每一个事务的提交也需要在内存中进行同步，不过在内存中这样的操作要比硬盘要快得多了。 如果你想了解更多文件通道的技术内幕，可以去看Brock Noland 的article about the Apache Flume File Channel。

使用大的Batch Size还有一个不好的地方，如果在事务传输中发生了失败，比如下游主机挂掉或者网络断开，会有一定的可能性发生数据重复。 举个例子，如果你的Batch Size是1000，而且你的下游机器离线了，那么可能就会发生大小为1000的数据重复。 这种情况的发生有几种可能，举个例子，如果事件已经写入了下游机器，在下游机器告知事务成功之前恰巧发生了连接断开，那么就会出现数据重复。然而你并不太需要忧心这一点，因为这只会发生在极其特殊的情况下。

----


#### 选择一个适当的Batch Size

为了尽可能的压榨Flume系统的性能， Batch Size 多次实验并小心选择。

再详细探讨Batch Size的选择之前， 下面有几条原则请大家参考：

* Sink的 Batch Size 应该等于输入流 Batch Size 的和。举个例子，如果你有一个 flume agent， 上游的10个Flume agent通过avro以100的batch Size 向这个agent发送事件，那么这个agent的batch Size 就应该设置为10X100 = 1000。

* 如果你发现你需要将Batch Size设置的非常大，比如比10000还大，那么需要考虑多设置几个channel来提高并发性（每一个channel都有一个自己的线程）。 实践中， 使用一个batch Size 为20000的HDFS Sink的性能要比 使用 两个batch Size 为10000的channel 性能要差。

* 在性能可接受的情况下，尽量选择较小的Batch Size。

* 为了做更详细的性能调优，要对channel做一些监控，例如Channel Size参数。（这个会在下一篇文章中讲到）


----


#### 不同情况下的不同Batch Size

由于Batch Size涉及到 性能/重复的权衡，我通常在不同应用场景下选用不同的batch Size。比如使用Flume 的HBase Sink ，经常采用100的Batch Size来降低系统的延迟。 而使用HDFS Sink的时候， 我就见到许多人使用10000的Batch Size来提高系统的吞吐，因为他们可以在接下来的MapReduce将重复的数据剔除掉。 值得注意的是， 当使用HDFS Sink且用以一个比较大的Batch Size的时候， 一些其他的参数也需要相应的提高。其中一个参数就是hdfs.callTimeout， 它或许需要提高到60s或更高来预防HDFS偶尔的慢响应。

另一方面，为了获取最佳性能，实践上会在收集端将Batch Size配置为1，channel也会配置为内存通道。并在下游的Flume Agent中配置比较大的Batch Size和文件通道，来获取更好的性能和可靠性保证。 为了获得更好的性能， 任意一层（包括应用层）都可以做一些尽可能的批量处理。

#### 配置参数与陷阱

实际上我们在sink端配置的都是Batch Size，但是它们不一定叫同一个名。 对于HDFS来说，它的实际叫hdfs.batchSize（由于一些历史原因）。 为了让大家更好理解，我曾经建议叫hdfs.txnEventMax 。 由于历史原因，HDFS Sink，从单个事务中提取的事件数和单次写入到HDFS的事件数不是同一个参数。实践上，确实有一些小原因这两个参数不应该设置为同一个值。

一个可能令人迷惑的陷阱就是sink 的 Batch Size 必须小于或等于  相应Channel 的  transactionCapacity . transactionCapacity 应该设置为从这个channel中一次提取事件的最大值。

下面是一个表格， 包括了Sink类型 、 配置参数 和 典型值。




| Sink Type      | Config parameter                 | Typical value |
| ----           | :---:                            | ----:         |
| Avro           | batch-size                       | 100           |
| HDFS           | hdfs.batchsize, hdfs.txnEventMax | 1000          |
| HBaseSink      | batchSize                        | 1000          |
| AsyncHBaseSink | batchSize                        | 100           |

