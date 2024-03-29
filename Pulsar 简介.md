# Kafka VS Pulsar
## 消息存储
### Kafka 中以分区为中心的存储
在消息传递系统中使用基于分区的策略时，Topic 被分成固定数量的相等大小的分组，称为分区。发布到主题的数据均匀分布在各个分区中，如下图所示，每个分区接收发布到主题的消息的三分之一。Topic 的总存储容量等于 Topic 中分区数乘以每个分区的大小。一旦达到此限制，就无法向主题添加更多数据。简单地向集群添加更多代理并不能缓解这个问题，因为您还需要增加必须手动执行的主题中的分区数量。此外，增加分区数量还需要执行重新平衡。
​

在以分区为中心的基于存储的系统中，分区的数量是在创建主题时指定的，因为这允许系统确定哪个节点将负责存储哪个分区等。但是，预先确定数量分区有一些意想不到的副作用：

- 单个分区只能存储在集群内的单个节点上，因此分区的大小受限于该节点上的可用磁盘空间量。
- 由于数据均匀分布在所有分区中，因此每个分区都限制为 Topic 中最小分区的大小。例如，如果一个topic 分布在3个节点上，分别有4TB、2TB和 1TB 的空闲磁盘，那么第三个节点上的分区大小只能增长到1TB，这意味着主题中的所有分区只能增长也到 1TB。
- 虽然不是严格要求，但每个分区通常会多次复制到不同的节点以确保数据冗余。因此，最大分区大小进一步限制为最小副本的大小。

​

![](https://cdn.nlark.com/yuque/0/2021/png/21680022/1631107112773-3390d537-caee-4c6e-bcd7-0fc80a93ff3d.png#clientId=ufff48177-a1a5-4&from=paste&id=ud3d468ee&margin=%5Bobject%20Object%5D&originHeight=544&originWidth=654&originalType=url&ratio=1&status=done&style=none&taskId=u60a6ca99-1588-421e-a395-ae72b794911)
遇到这些容量限制 ，则是通过增加主题的分区数进行补救。但是，在容量扩展过程需要重新平衡整个主题，如下图所示。在此平衡的过程中，现有 Topic 的数据将在所有 Topic 分区之间重新分配，以释放现有节点上的磁盘空间。因此，当您将第四个分区添加到现有 Topic 时，一旦重新平衡过程完成，每个分区应该有大约 25% 的消息总数。
这种数据的重新复制成本高昂且容易出错，因为它消耗的网络带宽和磁盘 I/O 与 Topic 的大小成正比，例如，重新平衡 10TB 的 Topic 将导致从磁盘读取 10TB 的数据，通过网络传输，并写入目标代理上的磁盘。只有在重新平衡过程完成后，才能删除之前存在的数据，Topic 才能恢复为客户端服务。因此，明智地选择分区大小是明智的，因为重新平衡的成本不能轻易消除。
![](https://cdn.nlark.com/yuque/0/2021/png/21680022/1631107390202-1330e74b-e768-409a-a35e-7bbc13915bc4.png#clientId=ufff48177-a1a5-4&from=paste&id=u1a203d56&margin=%5Bobject%20Object%5D&originHeight=585&originWidth=637&originalType=url&ratio=1&status=done&style=none&taskId=u79da5673-c10d-46f6-a42d-cc2ac3b77a3)
### Pulsar 中以 segment 为中心的存储
Pulsar 依靠 Apache BookKeeper 项目来提供其消息的持久存储。BookKeeper 的逻辑存储模型基于无限流条目存储为顺序日志的概念。从下图中可以看出，在 BookKeeper 中，每个日志都被分解为更小的数据块，称为segment，这些数据块又由多个日志条目组成。然后将这些段写入存储层中称为 bookie 的多个节点，以实现冗余和扩展。
![](https://cdn.nlark.com/yuque/0/2021/png/21680022/1631109293571-8094e83c-e851-4ec4-8381-d7b487e9d2e7.png#clientId=ufff48177-a1a5-4&from=paste&id=u9ebcfa88&margin=%5Bobject%20Object%5D&originHeight=543&originWidth=716&originalType=url&ratio=1&status=done&style=none&taskId=u3333ac06-3315-421a-9291-ff8c304dfed)
从上图中可以看出，segment 可以放置在存储层上具有足够磁盘容量的任何位置。当存储层中没有足够的存储容量用于新段时，可以轻松添加新节点并立即用于存储数据。以 segment  为中心的存储架构的主要优势之一是真正的水平可扩展性，因为 segment  可以无限地创建并存储在任何地方。与以分区为中心的存储不同，后者基于分区数量对垂直和水平扩展施加了人为限制。
​

## 消息消费
### Kafka 重的消息消费
在 Kafka 中，所有消费者都属于所谓的消费者组，它形成了一个 Topic 的单个“逻辑订阅者”。每个组由许多消费者实例组成，以实现可扩展性和容错性，如果一个实例失败，其余的消费者将接管。默认情况下，每当应用程序订阅 Kafka Topic 时都会创建一个新的消费者组，应用程序也可以通过提供 group.id 来利用现有的消费者组。


根据 Kafka 文档，“在 Kafka 中实现消费的方式是将日志中的分区划分到消费者实例上，以便每个实例在任何时间点都是分区“公平份额”的唯一消费者。” 通俗地说，**这意味着一个主题中的每个分区一次只能有一个消费者，并且这些分区在组内的消费者之间均匀分布。**如下图所示，如果一个消费者组的成员少于分区，那么一些消费者会被分配到多个分区，但是如果你的消费者比分区多，那么多余的消费者将保持空闲状态，只有在发生异常时才会接管消费者失败。
​

创建“独占”消费者的一个重要副作用是，在一个消费者组内，活跃消费者的数量永远不会超过 Topic 中的分区数量。这种限制可能会带来问题，因为从 Kafka Topic 扩展数据消费的唯一方法是向消费者组添加更多消费者。这有效地限制了分区数量的并行量，这反过来又限制了在您的消费者无法跟上 Topic 生产者的情况下扩展数据消耗的能力。不幸的是，唯一的补救方法是增加主题分区的数量，正如我们之前讨论的那样，这不是一个简单、快速或廉价的操作。
![](https://cdn.nlark.com/yuque/0/2021/png/21680022/1631109773152-21988571-cc3c-4133-9baa-99cb261e5861.png#clientId=ufff48177-a1a5-4&from=paste&id=u1497fd25&margin=%5Bobject%20Object%5D&originHeight=398&originWidth=644&originalType=url&ratio=1&status=done&style=none&taskId=u8be85ff3-e095-40d5-b015-325584e95c5)
您还可以从上图中看到，所有单个消费者的消息都被合并并发送回 Kafka 客户端。因此，消息排序不是由消费者组维护的。Kafka 仅提供分区内记录的总顺序，而不提供 Topic 中不同分区之间的总顺序。
​

## 数据持久化
### Kafka 中的数据持久化
Apache Kafka 采用以分区为中心的存储方法来存储消息。为了保证数据的持久性，集群内维护了每个分区的多个副本，以提供可配置级别的数据冗余。
![](https://cdn.nlark.com/yuque/0/2021/png/21680022/1631110450274-70e573e3-86b9-45ba-9c9e-d520a2f60dcf.png#clientId=ufff48177-a1a5-4&from=paste&id=u84005639&margin=%5Bobject%20Object%5D&originHeight=431&originWidth=694&originalType=url&ratio=1&status=done&style=none&taskId=uc873426b-d254-4a46-a02f-aa97e66010f)
当 Kafka 收到传入消息时，会对消息应用散列函数以确定消息应写入到主题的哪个分区。一旦确定，消息内容将写入分区“领导者”副本的页面缓存（而不是磁盘）。一旦消息被领导者确认，每个“跟随者”副本负责以拉取方式从分区领导者那里检索消息内容，如上图所示，即它们充当消费者并从分区领导者那里读取消息。领导者。这种整体方法就是所谓的最终一致性策略，其中分布式系统中有一个节点具有最新的数据视图，该节点最终会与其他节点通信，直到它们都实现数据的一致视图. 虽然这种方法的优点是可以减少存储传入消息所需的时间，但它也带来了两种数据丢失的机会；

- 如果领导节点发生断电或其他进程终止事件，任何写入页面缓存但尚未持久化到本地磁盘的数据都将丢失。
- 前的领导进程失败并且剩余的另一个追随者被选为新的领导者时。

在leader故障转移场景中，前一个leader确认但尚未复制到新当选leader副本的任何消息也将丢失。
​

这种复制策略的另一个副作用是只有领导者副本可以为生产者和消费者提供服务，因为它是唯一保证拥有最新和正确数据副本的副本。所有跟随者副本都是被动节点，无法在流量高峰期间减轻领导者的任何负载。
​

### Pulsar 中的数据持久性
当 Pulsar 收到一条传入消息时，它会在内存中保存一个副本，并将数据写入预写日志 (WAL)，在确认被发送回消息发布者之前，它会被强制写入磁盘，如图下图所示。这种方法是仿照传统的数据库 ACID 事务语义建模的，可确保即使机器出现故障并在将来重新上线，数据也不会丢失。
​

一个 Topic 所需的副本数量可以根据您的数据复制需求在 Pulsar 中配置，并且 Pulsar 保证在将确认发送给生产者之前，数据已被仲裁服务器接收并确认。这种设计确保只有在写入数据的所有 bookie 节点上同时发生致命错误的极不可能的事件中才会丢失数据。这就是为什么建议将 bookie 节点分布在多个区域并使用 rack-aware placement policies 来确保数据副本存储在多个区域或数据中心的原因。
​

![](https://cdn.nlark.com/yuque/0/2021/png/21680022/1631111007032-8cedfcb0-0c20-4292-b67a-dcdbccd4225a.png#clientId=ufff48177-a1a5-4&from=paste&id=ue0f96be8&margin=%5Bobject%20Object%5D&originHeight=479&originWidth=702&originalType=url&ratio=1&status=done&style=none&taskId=uaac708c3-d94c-4cd6-99c6-a9ca4caef04)
更重要的是，这种设计消除了对负责确保数据在副本之间保持同步的辅助复制过程的需要，并消除了由于复制过程中的任何滞后而导致的任何数据不一致问题。
​

​

## 消息确认
### Kafka 中的消息确认
恢复点在 Apache Kafka 中称为消费者偏移量，它完全由消费者控制。通常，消费者在从 Topic 读取记录以指示消息确认时以顺序方式增加其偏移量。但是，将此偏移量仅保留在消费者内存中是危险的。因此，这些偏移量也作为消息存储在名为“__consumer_offsets”的单独主题中。每个消费者定期向该主题提交一条包含其当前位置的消息，如果您使用 Kafka 的自动提交功能，则为每五秒一次。虽然这种策略比仅将偏移量保留在内存中要好，但这种定期更新方法会产生一些后果。
​

Kafka 消费者 API 提供了一种方法，可以在对应用程序开发人员有意义的点而不是基于计时器提交当前偏移量。因此，如果你真的想消除重复的消息处理，你可以使用这个 API 在每条成功消费消息后提交偏移量。然而，这将确保准确恢复偏移量的负担推给了应用程序开发人员，并给消息消费者带来了额外的延迟，他们现在必须将每个偏移量提交到 Kafka 主题并等待确认。
​

### Pulsar 中的消息确认
Apache Pulsar 在 Apache BookKeeper 内部为每个订阅者维护一个 ledger，称为用于跟踪消息确认的 cursor ledger 。当消费者阅读并处理了一条消息时，它会向 Pulsar Broker 发送确认。收到此确认后，Broker 立即更新该消费者订阅的 cursor ledger。由于此信息存储在 BookKeeper 的 ledger 中，我们知道它已被 fsync-ed 到磁盘，并且多个副本存在于多个 bookie 节点中。将此信息保存在磁盘上可确保消费者不会再次收到消息，即使他们在稍后的时间点崩溃并重新启动。
​

在 Apache Pulsar 中，有两种方式可以确认消息，选择性或累积。通过累积确认，消费者只需要确认它收到的最后一条消息。主题分区中直到并包括给定消息 ID 的所有消息都将被标记为已确认，并且不会再次重新传递给消费者。累积确认实际上与 Apache Kafka 中的偏移更新相同。
​

Apache Pulsar 与 Kafka 的不同之处在于消费者能够单独确认消息，也就是选择性确认。此功能对于支持每个主题的多个消费者至关重要，因为它允许在单个消费者发生故障时重新传递消息
​

## 消息保留
### Kafka 中的消息确认
Kafka 会在可配置的保留期限内保留发布到主题的所有消息。例如，如果保留策略设置为 7 天，则在消息发布到主题后的 7 天内，它可以立即使用。保留期过后，该消息将被丢弃以释放空间。此删除与消息是否已被消费和确认无关。显然，如果保留期小于所有消费者消费消息所需的时间，例如消费系统的长期中断，这将带来数据丢失的机会。


这种基于时间的方法的另一个缺点是，您保留消息的时间很可能比所需的时间长得多，即在所有相关消费者都使用它们之后，这是对存储容量的低效使用。
​

### Pulsar 中的消息保留
在 Pulsar 中，当消费者成功处理一条消息时，它需要向代理发送确认，以便代理可以丢弃该消息。默认情况下，Pulsar 会立即删除所有主题的消费者已确认的所有消息，并将所有未确认的消息保留在消息 backlog 中。在 Pulsar 中，只有在所有订阅都已经消费完消息后才能删除消息。通过配置消息保留期，Pulsar 还允许您将消息保留更长的时间，即使在所有订阅都已经消费了它们之后。
