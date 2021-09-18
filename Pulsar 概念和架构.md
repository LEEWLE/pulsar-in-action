# Pulsar 的物理架构
三个不同的地理区域中的每一个都会有一个 Pulsar 集群，总共三个 Pulsar 集群。其他消息传递系统从管理和部署的角度将集群视为最高级别，这需要将每个集群作为独立系统进行管理和配置。Pulsar 提供了更高级别的抽象，称为Pulsar Instance，它由一个或多个 Pulsar 集群组成，这些集群一起作为一个单元运行，可以从单个位置进行管理。
![](https://cdn.nlark.com/yuque/0/2021/png/21680022/1631147060790-382d5f93-9705-447a-814f-4a6b6537db12.png#clientId=u11661850-494d-4&from=paste&id=u88f7d7b6&margin=%5Bobject%20Object%5D&originHeight=185&originWidth=542&originalType=url&ratio=1&status=done&style=none&taskId=u68e0c87f-351f-4529-8f6f-b5c0a7294c1)
使用 Pulsar 实例的最大原因之一是启用异地复制。事实上，只有同一实例内的集群才能配置为在它们之间复制数据。
​

Pulsar 实例使用一个称为配置存储的实例范围的 Zookeeper 集群来保留与多个集群相关的信息，例如地理复制和租户级安全策略。这允许您在一个位置定义和管理这些策略。为了为配置存储提供弹性，Pulsar Instance 的 ZK 集成中的每个节点都应该跨多个区域部署，以确保其在区域故障等情况下的可用性。

需要注意的是，即使启用了异地复制，单个 Pulsar 集群也需要 Pulsar 实例用于配置存储的 Zookeeper 集合的可用性才能运行。启用异地复制后，如果配置存储关闭，则发布到相应集群的消息将在本地缓冲，并在集成再次运行时转发到其他区域。
​

## Pulsar 分成架构
在每个地理区域内都会有一个 Pulsar 集群，它由一个基于一个或多个 Pulsar 消息代理的无状态服务层组成；以及一个基于一个或多个 BookKeeper bookies的有状态存储层。
![](https://cdn.nlark.com/yuque/0/2021/png/21680022/1631147640805-e47871bb-9b73-4f43-8bc7-1026c518429c.png#clientId=u11661850-494d-4&from=paste&id=ucd0dfcc6&margin=%5Bobject%20Object%5D&originHeight=392&originWidth=651&originalType=url&ratio=1&status=done&style=none&taskId=ue637fcc4-e444-4e7c-b059-791ecc24c20)


当客户端访问尚未使用的 Topic 时，会触发一个流程来选择最适合获取该 Topic “所有权”的 Broker。一旦 Broker 获得某个 Topic 的所有权，它就负责处理对该 Topic 的所有请求，并且任何希望发布或使用该主题数据的客户端都需要与“拥有”它的相应 Broker 进行交互。因此，如果要将数据发布到特定 Topic ，则需要知道哪个 Broker 拥有该 Topic  并连接到它。但是，Broker 分配信息仅在 Zookeeper 元数据中可用，并且会根据负载重新平衡、代理崩溃等发生变化。因此，您无法直接连接到 Broker 本身，并希望与您想要的 Broker 进行通信。这正是 Pulsar Proxy 被创建的原因，它作为 Broker 集群中的一个中介存在。
​

### Pulsar Proxy
如果您将 Pulsar 集群托管在私有和/或虚拟网络环境（例如 Kubernetes）中，并且您希望提供与 Pulsar Broker 的入站连接，那么您需要将其私有 IP 地址转换为公共 IP 地址。虽然这可以使用传统的负载均衡技术和物理负载均衡器、虚拟 IP 地址或基于 DNS 的负载平衡等技术将客户端请求分布到一组代理中来实现，但这并不是提供冗余和故障转移的最佳方法。


传统的负载均衡器方法效率不高，因为负载均衡器不知道将哪个 broker 分配给给定 Topic ，而是将请求定向到集群中的随机 broker 。如果 broker 收到对其未提供服务的 Topic 的请求，它会自动将请求重新路由到适当的broker 进行处理，但这会导致时间上的重大损失。这就是为什么建议使用 Pulsar Proxy 代替它作为 Pulsar Brokers 的智能负载均衡器。
​

使用 Pulsar 代理时，所有客户端连接将首先通过 proxy 而不是直接到达 broker 本身。然后 broker 将使用 Pulsar 的内置服务发现机制来确定哪个 broker 托管您尝试访问的主题，并自动将客户端请求路由到它。此外，它将把此信息缓存在内存中，以供将来的请求简化查找过程。出于性能和故障转移目的，建议在传统负载均衡器后面运行多个 Pulsar Proxy。与 Brokers 不同，Pulsar Proxies可以处理任何请求，因此它们可以毫无问题地进行负载均衡。
​

### Stateless Serving Layer
Pulsar 的多层设计确保消息数据存储与 Brokers 分离，保证任何 Broker 可以随时为任何 Topic 传来的数据提供服务。 还允许集群随时将 Topic 的所有权分配给集群中的任何 Broker，这与将代理和它们所服务的主题数据并置在一起的其他消息传递系统不同。因此，我们使用术语“无状态”来描述服务层，因为 Broker 本身没有存储处理客户端请求所必需的信息。Broker 的“无状态”性质不仅允许我们动态扩展，还可以根据需要关闭，这也使集群能够适应多个 Broker 故障。 最后，Pulsar 有一个内部负载平衡机制，可以根据不断变化的消息流量在所有活动 Broker 之间持续重新平衡负载。
​

#### Bundles
将 Topic 分配给特定 Broker 是在所谓的捆绑级别完成的。 Pulsar 集群中的所有 Topic 都分配给了一个特定的 bundle，每个 bundle 都分配给了一个不同的 broker，如图 2.3 所示。 这有助于确保 a 中的所有 Topic 均匀分布在所有 Broker s中。
![image.png](https://cdn.nlark.com/yuque/0/2021/png/21680022/1631170042328-f99d89fe-96e4-46b2-820e-5075cf7a7c3d.png#clientId=u73009221-c7db-4&from=paste&id=ub5a4190c&margin=%5Bobject%20Object%5D&name=image.png&originHeight=272&originWidth=567&originalType=binary&ratio=1&size=14532&status=done&style=none&taskId=ue42b9a1d-b48c-480f-9e68-4d15ce8181b)
从服务的角度来看，每个 Broker 都被分配了一组包含多个 Topic 的 Bundle。Bundle 分配是通过散列 Topic 名称来确定的，这使我们能够确定它属于哪个 Bundle，而无需在 Zookeeper 中保留该信息。


每个 namespace 创建的 Bundle 数量由 Broker 配置文件中的 defaultNumberOfNamespaceBundles 属性控制，该属性的默认值为 4。若想为每个 namespace 提供不同的值，可以通过 Pulsar admin API 进行创建时指定值，会覆盖在 Broker 配置文件中的值。通常，您希望 Bundles 的数量是 Broker 数量的倍数，以确保它们均匀分布。 例如，如果每个 namespace 有 3 个 Broker 和 4 个 Bundle，那么其中一个 Broker 将被分配两个 Bundle，而其他 Broker 只得到一个。


#### Load Balance


虽然最初消息流量可能会尽可能均匀地分布在活动 Broker 上，但有几个因素可能会随着时间的推移而发生变化，从而导致负载变得不平衡。 消息流量模式的变化可能会导致 Broker 服务于多个流量较大的 Topic，而其他Topic 根本没有被使用。 当现有 Bundle 超过 Broker 配置文件中以下属性定义的某些预配置阈值时，该 Bundle 将被拆分为两个新 Bundle，其中一个被卸载到新 Broker ：


- loadBalancerNamespaceBundleMaxTopics 
- loadBalancerNamespaceBundleMaxSessions
- loadBalancerNamespaceBundleMaxMsgRate
- loadBalancerNamespaceBundleMaxBandwidthMbytes



该机制通过将这些过载的 Bundle 一分为二来识别和纠正某些 Bundle 比其他 Bundle 承受更重负载的情况。 然后可以将这些 Bundle 之一卸载到集群中的不同 Broker。
​

​

#### Load Shedding
Pulsar Brokers 有另一种机制来检测特定 Broker 何时过载，并自动让它卸载或卸载它的一些 Bundle 到集群中的其他 Broker。 当 Broker 的资源利用率超过由 broker 配置文件中的 loadBalancerBrokerOverloadedThresholdPercentage 属性定义的预配置阈值时，Broker 会将一个或多个 Bundle 卸载到新的 Broker。 此属性定义 Broker 可以消耗的可用 CPU、网络容量或内存的最大百分比。 如果这些资源中的任何一个超过此阈值，则触发卸载。
​

被选择的 bundle 保留，只是分配给不同的 Broker。 这是因为减载过程解决了与负载平衡过程不同的问题。 通过负载平衡，我们正在更正 Topic 在 Bundle 中的分布，因为其中一个比其他 Bundle 的流量大得多，我们正试图将负载分散到所有 Bundle 中。
​

另一方面，Load shedding 根据为它们提供服务所需的资源数量来更正跨 Broker 的 bundle 分布。 即使可以为每个 Broker 分配相同数量的 bundle，但如果 bundle 之间的负载不平衡，则每个 Broker 处理的消息流量可能会大不相同。
​

为了说明这一点，请考虑有 3 个 Brokers 和总共 60 个 Bundle 的场景，每个 Broker 服务 20 个 bundle。 此外，其中 20 个包目前处理了总消息流量的 90%。 现在，如果这些 Bundle 中的大多数恰好分配给同一个 Broker，则很容易耗尽该 Broker 的 CPU、网络和内存资源。 因此，将这些 Bundle 中的一些卸载到另一个 Broker 将有助于缓解问题，而拆分 Bundle 本身只会减少大约一半的消息流量，而将 45% 的消息流量留在原始 Broker 上。
​

#### Data Access Patterns


流系统中通常有三种 I/O 模式：

- Writes：将新数据写入系统；
- Tailing Reads：消费者在发布消息后立即读取最近发布的消息，
- Catch-up Reads：消费者为了 catch up 从 Topic 开始读取大量消息，例如当新的消费者想要访问从比最新消息早得多的时间点开始的数据时。

![image.png](https://cdn.nlark.com/yuque/0/2021/png/21680022/1631179812236-0f95cc88-dd3b-43d7-874a-2392ff4e2fbf.png#clientId=u73009221-c7db-4&from=paste&height=403&id=u91e58f36&margin=%5Bobject%20Object%5D&name=image.png&originHeight=537&originWidth=935&originalType=binary&ratio=1&size=115361&status=done&style=none&taskId=u1f38ee3b-476a-48c7-b463-ed8b362bcbe&width=701)


当生产者向 Pulsar 发送消息时，它会立即写入 BookKeeper。 一旦 BookKeeper 确认数据已提交，Broker 会在向生产者确认消息发布之前在其本地缓存中存储消息的副本。 这允许 Broker 直接从内存中读取最近发布的消息（Tailing Reads），并避免与磁盘访问相关的延迟。
​

当通过 “Catch-up Reads” 访问存储层数据的时，它变得更有趣。 当客户端消费来自 Pulsar 的消息时，该消息将经历上图所示的步骤。 Catch-up Reads 最常见的例子是当消费者长时间离线后再次开始消费时，即任何不直接从 Broker 的缓存中拿取数据的场景都将被视为 Catch-up Read， 例如将 Topic 重新分配给新的 Broker。


### Stream Storage Layer


Pulsar 保证所有消息消费者的消息传递。 如果消息成功到达 Pulsar Broker，它一定会被传送到其预定目标。 为了提供这种保证，所有未被确认的消息必须被持久化，直到它们可以被传递给消费者并被消费者确认。 正如我之前提到的，Pulsar 使用称为 Apache BookKeeper 的分布式 write-ahead log  (WAL) 系统来进行持久消息存储。 BookKeeper 按顺序提供日志流 entries 的持久存储，该服务被称为 ledgers。


#### Logical Storage Architecture


Pulsar 主题可以被认为是无限的消息流，它们按照接收消息的顺序依次存储。 传入的消息附加到流的末尾，而消费者则根据我之前讨论的数据访问模式在流的更上游读取消息。 虽然这种简化的视图让我们很容易推理消费者在Topic 中的位置，但由于存储设备的空间限制，这种抽象在现实中是不可能存在的。 最终，这个抽象的无限流概念必须在存在这种限制的物理系统上实现。


在实现流存储方面，Apache Pulsar 采用了与传统消息系统（例如 Kafka）截然不同的方法。 在 Kafka 中，每个流都被分成多个副本，每个副本都完全存储在 Broker 的本地磁盘上。 这种方法的优点在于它简单快速，因为所有写入都是顺序的，这限制了访问数据所需的磁盘磁头移动量。 Kafka 方法的缺点是单个 Broker 必须有足够的存储容量来保存分区数据，正如我在第 1 章中讨论的那样。


那么 Apache Pulsar 的方法有什么不同呢？ 对于初学者来说，每个 Topic 都不是以分区的集合而构建，而是一系列的 segments。 这些 segments 中的每一个都可以包含固定数量的消息，默认值为 50,000。 一旦一个 segments 已满，就会创建一个新 segments 来保存新消息。 因此，一个 Pulsar Topic 可以被认为是一个无界的 segments 列表，每个 segments 包含一个消息子集，如下图所示，它显示了流存储层的逻辑架构以及它如何映射到底层物理实现


![image.png](https://cdn.nlark.com/yuque/0/2021/png/21680022/1631181758545-6e51d4be-c4e7-4384-b0f5-dec2d794dab1.png#clientId=u73009221-c7db-4&from=paste&id=u02d6d8d7&margin=%5Bobject%20Object%5D&name=image.png&originHeight=412&originWidth=588&originalType=binary&ratio=1&size=41484&status=done&style=none&taskId=u51674b91-34b3-4818-8f33-1b3a012df89)
Pulsar Topic 的数据存储为 BookKeeper 层内的一系列 ledgers。 这些 ledgers ID 的列表存储在 Zookeeper 上称为 managed ledger 的逻辑结构中。 每个 ledger 包含 50,000 个 entries，其中包含存储跨多个 Bookies 的物理数据副本以备冗余。


Pulsar Topic 只不过是一个可寻址端点，用于唯一标识 Pulsar 中的特定 Topic，并且类似于 URL，因为它仅用于唯一标识客户端尝试连接的资源。 Topic 名称必须由 Pulsar Broker 解码以确定数据的存储位置。
​

Pulsar 在 BookKeeper 的 ledgers 之上添加了一个额外的抽象层，称为 managed ledgers，它保留了持有发布到 Topic 的数据的 ledger 的 ID。 正如我们上图中看到的，当数据第一次发布到 Topic A 时，它被写入到 Ledger-20。 在向该 Topic 发布 50,000 条记录后，ledger 被关闭，并创建了另一个 (ledger-245) 来代替它。 这个过程每 50,000 条记录存储传入的数据一次，并且 managed ledger 在 Zookeeper 内部保留了ledger ID 的这个唯一序列。
​

稍后当消费者尝试从 Topic A 读取数据时，managed ledger 用于定位 BookKeeper 内部的数据并将其返回给消费者。 如果消费者从最旧的消息开始执行 catch-up read，那么它将首先从 ledger-20 中获取所有数据，然后是 ledger-245 等。这些 ledgers 从最旧到最年轻的遍历对消费者来说是透明的，给用户造成单一顺序数据流的错觉。managed ledgers 允许这种情况发生并保留 BookKeeper ledgers的顺序，以确保以与发布相同的顺序读取消息。


#### Bookkeeper Physical Architecture
 在 BookKeeper 中，ledger 的每个单元都被称为一个 entry。 这些 entry 包含来自传入消息的实际原始字节以及用于跟踪和访问 entry 的一些重要元数据。 最关键的元数据是其所属 ledger  的 ID，该 ID 保存在本地 Zookeeper 实例中，以便将来消费者尝试读取消息时，可以从 BookKeeper 快速检索该消息。 log entries 流存储在称为 ledger  的仅附加数据结构中，如下图所示。
![](https://cdn.nlark.com/yuque/0/2021/png/21680022/1631197333779-8b57ee90-2f0d-4939-a9de-8845303a0e1a.png#clientId=u11661850-494d-4&from=paste&id=u952f9107&margin=%5Bobject%20Object%5D&originHeight=146&originWidth=354&originalType=url&ratio=1&status=done&style=none&taskId=u45e6b18b-cac7-487b-8d3e-c76d3422644)
Ledgers 具有仅附加语义，这意味着 entries 按顺序写入 ledger ，一旦写入 ledger  就无法修改。 从实践的角度来看，这意味着：

- Pulsar Broker 首先创建一个 ledger ，然后将 entries 附加到 ledger中，最后关闭分类帐ledger。 在此过程中不允许进行其他交互。
- 在 ledger 关闭后，无论是正常情况还是由于进程崩溃，它只能以只读模式打开。
- 最后，当 ledger  中的 entries  不再需要时，可以从系统中删除整个 ledger 。

​

负责存储 ledgers （更具体地说，ledgers 片段）的各个 BookKeeper 服务器被称为 Bookies。 每当 entries  写入ledger 时，这些 ledger 将写入一个称为集合的 bookies 节点子组。 集合的大小等于您为 Pulsar Topic 指定的replication factor (R)，并确保您将 entry 的 R 个副本保存到磁盘以防止数据丢失。
​

Bookies 以日志结构的方式管理数据，该方式使用三种类型的文件实现：journals、entry logs, 和 index file.。journal 保留所有 BookKeeper 事务日志。 在对 ledger  进行任何更新之前，bookie  确保将描述更新的事务写入磁盘以防止数据丢失。
​

![](https://cdn.nlark.com/yuque/0/2021/png/21680022/1631198235345-bec69ce9-4d32-4be2-9259-639132ee7cd8.png#clientId=u11661850-494d-4&from=paste&id=u68597843&margin=%5Bobject%20Object%5D&originHeight=475&originWidth=994&originalType=url&ratio=1&status=done&style=none&taskId=u1a497fa5-b83a-4e9f-a2ce-218dec9b63b)


Entry Log 包含写入 BookKeeper 的实际数据。 来自不同 ledgers 的 Entries  按顺序聚合和写入，而它们的偏移量作为指针保存在 ledger  缓存中以进行快速查找。 **为每个 ledger 创建一个索引文件，其中包含多个索引，记录存储在 Entry Log 中的数据的偏移量。** 索引文件以传统关系数据库中的索引文件为模型，并允许 ledger  消费者ledger  进行快速查找。 当客户端向 Pulsar 发布消息时，该消息将通过以下步骤（如上图所示）将其持久化到 BookKeeper ledger 中的磁盘。
​

通过将 entry  数据分布在不同磁盘设备上的多个文件中，bookies  能够将读取操作的影响与正在进行的写入操作的延迟隔离开来，从而允许他们处理数千个并发读取和写入。
​

### Metadata Storage


最后，每个集群还有自己的本地 Zookeeper 集成，Pulsar 使用它来存储集群特定配置信息，例如租户、命名空间和主题，包括安全和数据保留策略等。 这是我们早些时候讨论的 managed ledger 信息的补充。 
​

#### Zookeeper Basics  
根据Apache官网的说法，“ZooKeeper是一个集中式服务，用于维护配置信息、命名、提供分布式同步、提供组服务。”，这是对分布式数据源的详细说法。 Zookeeper 提供了一个分散的位置来存储信息，这在 Pulsar 或 BookKeeper 等分布式系统中至关重要。


Apache Zookeeper 解决了几乎每个分布式系统都必须解决的达成共识（即协议）的基本问题。 分布式系统中的进程需要就几个不同的信息达成一致，例如当前配置值、主题的所有者等。这对于分布式系统来说尤其是一个问题，因为同一组件有多个副本并发运行，没有真正的方法来协调它们之间的信息。 传统数据库不是一种选择，因为它们在框架内引入了一个序列化点，所有调用服务都被阻塞，等待表上的相同锁，这从本质上消除了分布式计算的所有好处。
​

访问共识实现使分布式系统能够通过提供比较和交换 (CAS) 操作来实现分布式锁，从而以更有效的方式协调进程。 CAS 操作将从 Zookeeper 检索到的值与预期值进行比较，并且仅当它们相同时才更新该值。 这保证了系统根据最新信息进行操作。 一个这样的例子是在写入任何数据之前检查 BookKeeper  ledger  的状态是否为 OPEN。 如果某个其他进程关闭了 ledger ，它将反映在 Zookeeper 数据中，并且该进程将知道不继续进行写入操作。 相反，如果一个进程要关闭 ledger ，则此信息将发送到 Zookeeper，以便它可以传播到其他服务，以便他们在尝试写入之前知道它已关闭。


Zookeeper 服务本身公开了一个类似文件系统的 API，以便客户端可以操作简单的数据文件（znodes）来存储信息。 这些 znode 中的每一个都形成类似于文件系统的层次结构。 在下面的部分中，我将检查 Zookeeper 中保留的元数据以及它的使用方式和使用对象，以便您可以自己确切了解为什么需要它。 最好的方法是使用与 pulsar 一起分发的 zookeeper-shell 工具
```shell
# 开启 Zookeeper shell
/pulsar/bin/pulsar zookeeper-shell
# 列出根级节点下的子节点
ls /
# Pulsar 使用的所有 znode 的输出
[admin, bookies, counters, ledgers, loadbalance, managed-ledgers, namespace, pulsar,
schemas, stream, zookeeper]
```
从上述指令中可以看出，在 Zookeeper 中为 Apache Pulsar 和 BookKeeper 创建了总共 11 个不同的 znode，根据它们包含的信息和使用方式，这些属于四个类别之一。
​

#### Configuration Data
第一类信息是 tenants, namespaces, schemas 等的配置数据。所有这些信息都是缓慢变化的信息，只有在用户通过 Pulsar administration API 创建或更新新集群、租户、命名空间、模式、安全策略、消息保留策略、复制策略和架构等内容才会进行更新 。 此信息存储在 znodes、/admin 和 /schemas 中。
​

#### Metadata Storage
所有 Topic 的 managed ledger 信息都存储在 /managed-ledgers znode 中，而 BookKeeper 使用 /ledgers znode 来跟踪当前存储在集群内所有 bookie 中的所有 ledgers 。
```shell
# managed ledger工具允许您按 Topic 名称查找 ledger
/pulsar/bin/pulsar-managed-ledger-admin print-managed-ledger --managedLedgerPath
/customers/orders/persistent/food-orders --zkServer localhost:2181 

# 这个 Topic 有两个 ledgers ，一个有 50K 个 entries  已关闭，另一个是打开的
ledgerInfo { ledgerId: 20 entries: 50000 size: 3417764 timestamp: 1589590969679}
ledgerInfo { ledgerId: 245 timestamp: 0}
```
正如上述命令中看到的那样，还有一个名为 pulsar-managed-ledger-admin 的工具，它允许您轻松访问 Pulsar 使用的 managed ledger 信息，以便从 BookKeeper 读取和写入数据。 在这种情况下，Topic 数据存储在两个不同的 ledgers 上； LedgerId-20 已关闭，包含 50,000 个条目，LedgerId-245 当前处于打开状态，并将在其中发布传入数据。
​

####  Dynamic Coordination Between Services
其余的 znodes 都用于跨系统的分布式协调，包括 /bookies 维护在 BookKeeper 集群中注册的 Bookies 列表，以及 proxy service 使用的 /namespace 来确定哪个 Broker“拥有”给定主题。 从清单 2.3 中可以看出，/namespace znode 层次结构用于存储每个命名空间的 bundle ID。
​

```shell
# 开启 zookeeper-shell
/pulsar/bin/pulsar zookeeper-shell

# 每个租户有一个 znode
ls /namespace
[customers, public, pulsar]

# 每个命名空间有一个 znode
ls /namespace/customers
[orders]

# 每个 bundle_id 有一个 znode 
ls /namespace/customers/orders
[0x40000000_0x80000000]
get /namespace/customers/orders/0x40000000_0x80000000
{"nativeUrl":"pulsar://localhost:6650","httpUrl":"http://localhost:8080","disabled":false}
```
正如您在我们之前的讨论中所回忆的那样，proxy 对主题名称进行哈希处理以确定包名称，在本例中为 0x40000000_0x80000000。 然后代理查询 /namespace/{tenant}/{namespace}/{bundle id} znode 以检索“拥有”Topic 的 Broker 的 URL。
​

希望这能让你更深入地了解 Zookeeper 在 Pulsar 集群中扮演的角色，以及它如何提供一个服务，该服务可以被动态添加到集群的节点轻松访问，以便他们可以快速确定集群配置并开始处理客户端要求。 一个这样的例子是新添加的 Broker 能够通过引用 /managed-ledgers znode 中的数据开始从 Topic 提供数据。
​

## Pulsar 逻辑架构
我们的送餐平台将有多个服务，使用 Pulsar 消息传递平台在彼此之间发送和接收信息。 我们的应用程序关心司机位置服务以及发布司机的位置事件，我们使用这些事件来更新客户在途中订单的状态。 由于我们预计在高峰时段会有大量客户，因此我们可能会有成千上万的订阅司机位置向相关 Topic ，每个未完成的订单订阅一个Topic。
​

### Tenants, Namespaces, and Topics
在本节中，我们将介绍数据在集群内构建和存储的逻辑结构。 Pulsar 被设计为一个多租户系统，通过为每个部门提供安全和专有的消息传递环境，允许它在组织内的多个部门之间共享。 这种设计使单个 Pulsar 实例能够有效地充当整个企业的消息传递平台服务。 Pulsar 的逻辑架构通过 Tenant、Namespaces 和 Topic 的层次结构支持多租户，如下图所示。
​

![image.png](https://cdn.nlark.com/yuque/0/2021/png/21680022/1631241216080-3b3cdaea-5c10-4af2-916e-a80f60e4f334.png#clientId=u73009221-c7db-4&from=paste&id=u36592317&margin=%5Bobject%20Object%5D&name=image.png&originHeight=402&originWidth=692&originalType=binary&ratio=1&size=42375&status=done&style=none&taskId=u45f28f3b-640e-4fe2-909d-be1be1f317c)
#### Tenant
Pulsar 层次结构的顶部是租户，它们可以代表特定的业务部门、核心功能或产品线。 租户可以分布在集群中，每个租户都可以使用自己的身份验证和授权方案，从而控制谁可以访问存储其中的数据。 它们也是存储配额、消息生存时间和隔离策略的管理单元。


#### Namespace
每个租户可以有多个 namespaces，这是一种为了管理相关 Topic 的逻辑分组机制。 在 namespace 级别，您可以设置访问权限、fine-tune replication 设置、管理跨集群消息数据的异地复制以及控制 namespace 中所有Topic 的消息过期。
​

让我们考虑如何为我们的外卖应用程序构建 Pulsar 的 namespace。 为了隔离敏感的传入的支付数据，仅限于财务团队的成员可以访问，可以配置一个名为“EPayments”的单独租户，如图 2.1 所示，并应用访问策略，限制仅授予财务团队成员的可完全访问，以便他们可以执行审计和处理信用卡交易等。
​

在“E-Payments”租户中，您可以创建 namespace，一个名为“Payments”的 namespace 将保存包括信用卡支付和礼品卡兑换在内的收款，另一个命名为“Fraud Detection”，它包含那些被标记为可疑的交易以供进一步处理。 在这样的部署中，您可以将面向用户的应用程序限制为对“Payments” namespace 的只写访问权限，同时授予对欺诈检测应用程序的只读访问权限，以便它可以评估它们是否存在潜在的欺诈行为。
​

在 “Fraud Detection” namespace 中，您可以为欺诈检测应用程序配置写访问权限，因此它可以将潜在的欺诈性支付放入“Risk Score”主题中。 您还可以授予对同一名称 namespace 的电子商务应用程序的只读访问权限，以便它可以收到任何潜在欺诈的通知并做出相应的反应，例如阻止销售。


#### Topic
Topic 是 Pulsar 中唯一可用的通信通道类型，所有消息都写入 Topic 和从 Topic 读取。 其他消息传递系统支持不止一种通信通道类型，例如，Topic 和队列根据它们支持的消息消费类型进行区分。 正如我在第 1 章中讨论的，队列支持先进先出的独占消息消费，而 Topic 支持发布-订阅、一对多的消息消费。 Pulsar 没有这样的区分，而是依靠各种订阅类型来控制消息消费模式。
​

除非另有说明，Pulsar 中的 Topic 由一个 Broker 服务，该 Broker 负责接收和传递所有消息。 因此，单个 Topic 的吞吐量受限于为其提供服务的 Broker 的计算能力。


​

#### Partitioned Topics
Pulsar 还支持可以由多个 Broker 为 partitioned topics 提供服务的概念，当负载分布在多台机器上时，它允许更高的吞吐量。 在程序内部，partitioned topics 被实现为 N 个内部 Topic ，其中 N 是分区数。 跨 Broker 的分区分布由 Pulsar 自动处理，有效地使过程最终对用户透明。


将 partitioned topics 实现为一系列单独的 Topic 并允许用户增加分区的数量时无需重新平衡整个 Topic。 相反，将为新分区创建新的内部 Topic，并可将其立即用于接收新消息，而根本不会影响其他内部 Topic，例如，消费者仍然可以不间断地读取/写入现有分区的消息。


从消费者的角度来看，分区 Topic 和普通 Topic 之间没有区别。 所有消费者订阅的工作方式与它们在非分区主题上的工作方式完全相同。 将消息发布到分区 Topic 时发生的情况有很大不同。 消息生产者负责确定消息最终发布到哪个内部 Topic。 如果消息存在 key，那么生产者将对该 key 进行 hash 后确定要发布到哪个主题。 这确保了所有具有相同 key 的消息都存储在同一个 Topic 中，并且将按照它们发布的顺序进行。
​

当发布没有 key 的消息时，生产者应该配置一个路由模式，指定如何在跨 Topic 的分区中路由消息。 默认路由模式称为 RoundRobinPartition，它会以循环方式跨所有分区发布消息。 这种方法将消息均匀地分布在各个分区上，从而最大限度地提高了发布吞吐量。 或者，您可以使用 SinglePartition 路由模式，该模式随机选择单个分区将其所有消息发布到其中。 当您没有 key 值时，此方法可用于将来自特定生产者的消息分组在一起以维护消息排序。 如果需要对跨分区 Topic 的消息分发进行更多控制，您也可以提供自己的路由实现。


让我们看看图 2.9 中描述的消息流，其中生产者被配置为使用 RoundRobinPartition 发布模式。 在这种情况下，生产者连接到 Pulsar proxy 并期望返回分配给它正在写入的 Topic 的 Broker 的 IP 地址。 Proxy 从本地 Metastore 获取此信息，并发现 Topic 已分区，需要将指定的分区 Id 转换为为该分区提供服务的内部 Topic 的名称。


![image.png](https://cdn.nlark.com/yuque/0/2021/png/21680022/1631245861146-4c573451-5ecd-4f88-91f6-4b0399060776.png#clientId=u73009221-c7db-4&from=paste&height=354&id=u14fbd2cd&margin=%5Bobject%20Object%5D&name=image.png&originHeight=471&originWidth=758&originalType=binary&ratio=1&size=78802&status=done&style=none&taskId=ufd074bbd-27c2-4c73-b746-e69155d7a98&width=569)
**_Figure 2.9: Publishing to a Partitioned Topic_**


在图 2.9 中，生产者的 round-robin routing strategy 决定消息应该发布到分区号 3，该分区实现为内部 Topic “p3”。 Proxy 还可以确定内部 Topic “p3”当前由 Broker-0 提供服务。 因此，消息被路由到该代理并写入“p3” Topic。 由于路由模式是轮询，同一生产者的后续调用将导致消息被路由到 Broker-1 上的“p4”内部 Topic 等。
​

### Addressing Topics in Pulsar


Pulsar 逻辑层的层次结构反映在用于访问 Pulsar 中 Topic 的命名约定中。 如图 2.10 所示，Pulsar 中的每个 Topic 地址都包含它所属的 Tenant 和 Namespace。 该地址还包含一个持久性前缀，指示消息内容是持久化到长期存储中还是仅保留在 bookie 的内存空间中。 如果创建的 Topic 名称带有前缀“persistent://”，则所有已接收但尚未确认的消息将存储在多个 bookie 节点上，因此可以在 Broker 失败等情况下幸免于难。
​

Pulsar 还支持非持久性 Topic ，它将所有未确认的消息保留在 Broker 内存中。 非持久性 Topic 名称以“non-persistent://”前缀开头以指示此行为。 使用非持久性 Topic 时，Broker 会立即将消息传递给所有连接的订阅者，而不会持久化它们。
​

![](https://cdn.nlark.com/yuque/0/2021/png/21680022/1631315971391-4c86234c-013e-46c9-91d0-ee975d1a118d.png#clientId=u11661850-494d-4&from=paste&id=u94fe6b85&margin=%5Bobject%20Object%5D&originHeight=33&originWidth=421&originalType=url&ratio=1&status=done&style=none&taskId=u5c72ed3f-5bf7-4bce-8270-62c2cec4b6f)
**_ Figure 2.10: Topic Addressing Scheme in Pulsar  _**
**_​_**

使用非持久性交付时，任何形式的 Broker 故障或订阅者与 Topic 断开连接等 Broker  故障都意味着该（非持久性）Topic 上的所有传输中消息都将丢失，即使重新连接，订阅者也将永远无法接收这些消息。 虽然非持久性消息传递通常比持久性消息传递更快，因为它避免了与将数据持久化到磁盘相关的延迟，但只有在您确定您的用例可以容忍消息丢失时才建议使用它们。
​

​

### Producers, Consumers, and Subscription
Pulsar 建立在发布-订阅模式上，也就是 pub-sub。 在这种模式中，生产者向 Topic 发布消息。 然后消费者可以订阅这些 Topic ，处理传入的消息，并在处理完成时发送确认。
​

生产者是直接或通过  Pulsar proxy 连接到 Pulsar Broker 的任何进程，并将消息发布到 Topic。 消费者是连接到 Pulsar broker 以从 Topic 接收消息的任何进程。 当消费者成功处理一条消息时，它需要向 Broker 发送确认，以便 Broker 知道它已被接收和处理。 如果在预配置的时间范围内未收到此类确认， 然后 Broker 会将其重新交付给该订阅的消费者。
​

当消费者连接到 Pulsar Topic 时，它会建立所谓的“订阅”，它指定如何将消息传递给一组中一个或多个消费者。 Pulsar 有四种可用的订阅模式：独占、故障转移、Key Shared 和 Shared。 无论订阅类型如何，消息都按收到的顺序传送。
​

有关这些订阅的信息保留在本地 Pulsar Zookeeper 元数据中，并包括所有消费者的 http 地址等。 每个订阅还有一个与之相关联的游标，它表示为订阅消费和确认的最后一条消息的位置。 为了防止消息重新传递，这些订阅游标保留在 bookies  上，以确保它们在任何代理级故障时都能幸免于难。
​

Pulsar 支持每个 Topic 的多个订阅，这允许多个消费者从一个 Topic 读取数据。在图 2.11 中看到的，该 Topic 有两个不同的订阅， Sub-A 和 Sub-B。 消费者A 首先连接到 Topic 并以独占消费者模式运行，这意味着 Topic 中的所有消息都将由消费者A消费。 到目前为止，Consumer-A 只确认了前 4 条消息，因此 Sub-A 其订阅的游标位置当前设置为 5。
​

![](https://cdn.nlark.com/yuque/0/2021/png/21680022/1631316789721-c19bf1df-b19c-4051-b5ff-7040ff9c3313.png#clientId=u11661850-494d-4&from=paste&id=u69fbbe3e&margin=%5Bobject%20Object%5D&originHeight=234&originWidth=692&originalType=url&ratio=1&status=done&style=none&taskId=u1021977b-1b12-4781-a32c-b974b34ebb3)
**_Figure 2.11: Pulsar supports multiple subscriptions per topic, which allows multiple consumers to read the same data.  _**
​

名为 Sub-B 的订阅是在前 3 条消息生成后创建的，因此这些消息都没有传递给该订阅的使用者。 一个常见的误解是，在某个 Topic 上创建的任何订阅都将从该 Topic 的第一条消息开始，这就是为什么我选择在这里说明这一点，并表明您只会在订阅后收到发布到该 Topic 的消息。
​

我们还可以看到，由于 Sub-B 运行在共享模式下，消息已经分发到组中的所有消费者，每条消息仅由组中的单个消费者处理。 您还可以看到 Sub-B 的游标更领先于 Sub-A 的游标，这在您将消息分发到多个消费者时并不少见。
​

### Subscription Types


在 Pulsar 中，所有消费者都使用订阅来消费来自主 Topic 数据。 订阅只是定义如何将消息传递给给定 Topic 的消费者的配置规则。 Pulsar 订阅可以在多个应用程序之间共享，大多数订阅类型都是专门为这种使用模式设计的。 Pulsar 支持四种不同类型的订阅：独占、故障转移、Key Shared 和 Shared，如图 2.12 所示。


![](https://cdn.nlark.com/yuque/0/2021/png/21680022/1631317168221-3a2abc41-2779-4f80-8179-06396e9f9784.png#clientId=u11661850-494d-4&from=paste&id=ue738142c&margin=%5Bobject%20Object%5D&originHeight=397&originWidth=792&originalType=url&ratio=1&status=done&style=none&taskId=uc88a19fe-344c-48e4-896e-629a2c6f5d6)
**_ Figure 2.12 Pulsar’s Subscription Modes  _**
**_​_**

Pulsar 的每种订阅类型都服务于不同类型的用例，因此了解它们以正确使用它们很重要。 让我们重新审视这样一个场景，一家金融服务公司将实时股票市场报价信息传输到名为“stock quote”的 Topic 中，并希望在整个企业范围内共享该信息，看看这些订阅模式中的每一种将如何用于相同的用例。
**_​_**

#### Exclusive


独占订阅只允许单个消费者访问该订阅的消息。 如果任何其他消费者尝试使用相同订阅f方式订阅主题，则会引发异常并且无法连接。 当您要确保每条消息只由已知使用者处理一次时，可以使用此模式。
​

在我们的金融服务组织内，数据科学团队将使用这种类型的订阅，通过他们的机器学习模型来提供股票 Topic 数据，以便训练和/或验证它们。 这将允许他们完全按照收到的顺序处理记录，以按正确的时间顺序提供股票报价流。 每个模型都需要自己的独占订阅，如图 2.13 所示，以接收自己的数据副本。
​

![](https://cdn.nlark.com/yuque/0/2021/png/21680022/1631317482378-7e294e42-784f-4ff3-aa22-a6724d57d078.png#clientId=u11661850-494d-4&from=paste&id=u535dfd72&margin=%5Bobject%20Object%5D&originHeight=143&originWidth=690&originalType=url&ratio=1&status=done&style=none&taskId=ua8d9049e-bc58-4a96-9cb1-99aca7de58f) Figure _**2.13: An exclusive subscription only permits a single consumer to consume the messages.  **_




#### Failover Subscriptions


故障转移订阅允许多个消费者附加到订阅，但只选择一个消费者来接收消息。 此配置允许您提供故障转移使用者以在使用者发生故障时继续处理主题中的消息。 如果活动消费者无法处理消息，Pulsar 会自动故障转移到列表中的下一个消费者并继续传递消息。


当您希望具有消费者高可用性的单一处理语义时，这种类型的订阅非常有用。 如果您希望应用程序在系统出现故障时继续处理消息，并且在第一个使用者因任何原因失败时由另一个使用者接管，这将非常有用。 通常，这些使用者分布在不同的主机和/或数据中心，以确保应用程序可以在多次中断后幸免于难。 正如您在图 2.14 中所见，消费者 A 是活动消费者，而消费者 B 是备用消费者，如果消费者 A 因任何原因断开连接，它将成为下一个接收消息的消费者。
![](https://cdn.nlark.com/yuque/0/2021/png/21680022/1631317652962-305192ff-dec6-4e8f-87e2-eb64e694e0bd.png#clientId=u11661850-494d-4&from=paste&id=u5ea0a549&margin=%5Bobject%20Object%5D&originHeight=236&originWidth=690&originalType=url&ratio=1&status=done&style=none&taskId=u48e2786b-4170-4601-bcb9-24313e6d7d6)
_**Figure 2.14: A failover subscription has only one active consumer at a time; but permits multiple stand-by consumers.  **_
​

一个这样的例子是，如果我们金融服务公司的数据科学团队部署了他们的一个模型，该模型使用来自“stock quotes”Topic 的数据，该 Topic 生成市场波动率分数，这些分数与其他模型的分数相结合，以为交易团队产生总体建议。 至关重要的是，该模型的一个实例保持运行并始终运行以帮助交易团队做出明智的交易决策，运行多个实例并生成建议可能会影响整体建议。
​

#### Shared Subscriptions


共享订阅允许多个消费者附加到订阅，每个消费者都可以主动接收消息，这与一次仅支持一个活动消费者的故障转移订阅不同。 消息以循环方式传递给所有注册的消费者，任何给定的消息都只传递给一个消费者，如图 2.15 所示。


![](https://cdn.nlark.com/yuque/0/2021/png/21680022/1631317863510-19dca34f-2423-40b1-8665-baa9a0539c31.png#clientId=u11661850-494d-4&from=paste&id=u86b80c33&margin=%5Bobject%20Object%5D&originHeight=392&originWidth=680&originalType=url&ratio=1&status=done&style=none&taskId=u61489ccf-bcdd-4e00-863a-42560614b4c)
 **_Figure 2.15: Messages are distributed across all consumers of a shared subscription.  _**
**_​_**

这种订阅类型对于实现工作队列很有用，其中消息排序并不重要，因为它允许您快速扩展主题上的使用者数量以处理传入的消息。 每个共享订阅的消费者数量没有上限，这允许您通过增加消费者数量达到存储层的极限。


在我们虚构的金融服务组织中，我们的内部交易平台、算法交易系统和面向客户的网站等关键业务应用程序都将从此类订阅中受益。 这些应用程序中的每一个都将使用自己的共享订阅，如图 2.15 所示，以确保它们每个都收到发布到股票 Topic 的所有消息。


#### Key-Shared Subscriptions


key-shared 订阅也允许多个消费者，但与以循环方式在消费者之间分发消息的 shared  订阅不同，它添加了一个二级 key  索引，以确保具有相同 key  的消息被传递给同一个消费者 . 此订阅充当 SQL 中的 GROUP BY 语句，其中具有相似键的数据组合在一起。 这在您想在使用之前对数据进行预排序的情况下特别有用。
​

考虑业务分析团队需要对股票 Topic 中的数据执行一些分析的场景。 通过使用 key-shared 订阅，他们可以确保给定股票代码的所有数据都将由同一个消费者处理，如图 2.16 所示。 使他们更容易将这些数据与其他数据流连接起来。
​

![](https://cdn.nlark.com/yuque/0/2021/png/21680022/1631335874786-e8c9a859-df5b-457f-87ac-5cd258414e8e.png#clientId=u11661850-494d-4&from=paste&id=u9bde200d&margin=%5Bobject%20Object%5D&originHeight=234&originWidth=690&originalType=url&ratio=1&status=done&style=none&taskId=u60eb2143-f5b7-4cfb-882c-a679b67d4f8) 
**_Figure 2.16: Messages are grouped together by the specified key in a shared-key subscription.  _**


总之，独占订阅和故障转移订阅只允许每个订阅的每个 Topic 分区有一个消费者，以确保消息按照收到的顺序被消费。 它们最适用于需要严格排序的用例。
​

另一方面，Shared 订阅允许每个 Topic 分区有多个消费者。 订阅中的每个消费者仅接收发布到 Topic 的消息的一部分。 Shared 订阅最适合不需要严格消息排序但需要高吞吐量的排队用例。


### Message Retention and Expiration
作为一个消息传递系统，Pulsar 的主要功能是将数据从 A 点移动到 B 点。一旦数据已经交付给所有预定的接收者，则说明不需要保留它。 因此，Pulsar 中的默认消息保留策略正是这样做的，当一条消息发布到 Pulsar Topic 时，它将被存储，直到它被 Topic 的所有消费者确认，然后被删除。此行为由 broker  配置文件中的 `defaultRetentionTimeInMinutes` 和 `defaultRetentionSizeInMB` 属性进行控制，默认情况下这两个属性都设置为 0 以指示不应保留已确认的消息。


#### Data Retention
但是，Pulsar 还支持 Namespace 级别的保留策略，允许您在希望 Topic 数据保留更长时间的情况下覆盖此默认行为，例如，如果您想在以后的某个时间点通过 Reader  接口或 SQL 访问 Topic 中的数据。
​

这些保留策略决定了在消息被所有已知消费者确认消费后，您将消息保留在持久存储中的时间。 保留策略未涵盖的已确认消息将被删除。 保留策略由大小和时间限制的组合定义，并应用在 Namespace 中 每个 Topic 上。 例如。 如果您指定大小限制为 100GB ，则该 Namespace 内的每个 Topic 中将保留多达 100GB 的数据，一旦超过此大小限制，将从 Topic 中清除消息（从最旧到最新），直到总数据量低于指定限制。 同样，如果您指定时间限制为 24 小时，则 Namespace 中所有 Topic 的已确认消息将根据 Broker 收到的时间最多保留 24 小时。


保留策略要求您指定大小和时间限制，它们彼此独立应用。 因此，如果消息违反了这些限制中的任何一个，则无论它是否符合其他策略，它都会从 Topic 中删除。
​

如果您指定时间限制为 24 小时和大小限制为 10GB 的保留策略，如清单 2.4 所示的 E-payments/refunds Namespace，那么当达到指定的策略限制之一时，数据将被删除。 因此，如果总容量超过 10GB，则有可能删除 24 小时以内的消息，反之亦然。
​

```java
# Listing 2.4 Setting Various Pulsar Retention Policies

# 保留所有 24 小时以内的消息，没有大小限制
./bin/pulsar-admin namespaces set-retention E-payments/payments \
--time 24h \
--size -1
 
# 保留多达 20GB 的消息，不受时间限制
./bin/pulsar-admin namespaces set-retention E-payments/fraud-detection \
--time -1 \
--size 20G 

# 最多保留 10GB 的不到 24 小时的消息。
./bin/pulsar-admin namespaces set-retention E-payments/refunds \
--time 24h \
--size 10G

# 保留所有消息
./bin/pulsar-admin namespaces set-retention E-payments/gift-cards \
--time -1 \
--size -1 #D
```
​

可以在创建保留策略的时候设置 `defaultRetentionTimeInMinutes` 或 `defaultRetentionSizeInMB`  的值为 -1，这表示为该 Namespace 下的 Topic 的保留策略设置为无限大小或时间。因此，使用该策略时要小心，因为数据永远不会从存储层中删除，所以请确保您有足够的存储容量或配置了定期将数据卸载到分层存储。  
​

#### Backlog Quotas


Backlog 是用于 Topic 中所有未确认消息的术语，这些消息必须存储在 bookie 中，直到它们被传递给所有预期的接收者。 默认情况下，Pulsar 会无限期地保留所有未确认的消息。 但是，Pulsar 可以通过修改 Namespace 级别的 backlog quota 策略覆盖此行为，可以减少一个或多个消费者由于系统崩溃等原因长时间离线的情况下减少未确认消息占用的空间。
​

这些  backlog quotas 旨在解决 Topic 生产者发送的消息过多，导致消费者处理的消息不会落后太多。 在这种情况下，您会希望防止消费者落后太多以至于永远赶不上。 发生这种情况时，您需要考虑消费者正在处理的数据的及时性，并确保消费者放弃较旧的、转而处理的较新的消息。 如果您的 Topic 中的数据在长时间内处于“陈旧”状态，那么实施 backlog quota 将帮助您通过限制 backlog 的大小，将处理工作集中在较新的数据上。


![image.png](https://cdn.nlark.com/yuque/0/2021/png/21680022/1631412086445-90b77e1a-a4f5-4467-bf43-eab472097510.png#clientId=u4e323e79-230b-4&from=paste&id=u0772f4e4&margin=%5Bobject%20Object%5D&name=image.png&originHeight=137&originWidth=653&originalType=url&ratio=1&size=15912&status=done&style=none&taskId=ucccc1912-0edb-40d3-9871-9ba16c02635)
**_Figure 2.17: Pulsar’s Backlog Quota allows you to dictate what action the Broker should take when the volume of unacknowledged messages exceeds a certain size. This prevents the backlog from growing so large that the consumer is processing data that is of little or no value._**
**_​_**

与我在上一节中讨论的旨在延长 Pulsar Topic 内已确认消息的生命周期的消息保留策略不同，这些 backlog quota 策略旨在缩短未确认消息的生命周期。
​

您可以通过配置一个 backlog 策略来限制这些消息 backlog 的允许大小，该策略指定 Topic 的 backlog 的最大允许大小以及超过此阈值时要采取的操作，如图 2.17 所示。  backlog retention  策略有三个不同的选项，它们决定了 Broker 应该采取的行为来缓解这种情况：

- Broker 可以通过指定 producer_request_hold 保留策略来拒绝入站消息，向生产者发送异常以指示他们应该推迟发送新消息  
- 当指定了 producer_exception 策略时，Broker 不会要求生产者推迟，而是强制断开任何现有的生产者。
-  如果您希望 Broker 丢弃来自 Topic 的现有的、未确认的消息，那么您应该指定 consumer_backlog_eviction 策略。

​

每一种都为您提供了三种截然不同的方法来处理图 2.17 中所示的情况。 第一个，producer_request_hold，将使生产者保持连接状态，但抛出异常以减慢它的速度。 此策略适用于您希望客户端应用程序捕获抛出的异常并在稍后重新发送消息的场景。 因此，当您不想拒绝从消费者发送的任何消息并且客户端将在重新发送之前将被拒绝的消息缓冲一段时间时，最好使用此策略。
​

而第二个策略 producer_exception 会强制完全断开生产者的连接，这将阻止消息发布，但需要生产者代码检测这种情况并自动重新连接。 使用此策略，很可能会丢失客户端生产者在断开连接期间发送的消息。 当您知道生产者无法缓冲消息时，最好使用此策略，例如，它们在资源受限的环境（例如 IoT 设备）中运行，并且您不希望 Pulsar 无法接收消息导致客户端应用程序崩溃。
​

最后一个选项不会影响生产者的功能，它将继续以当前速率生成消息。 但是，尚未使用的旧消息将被丢弃，从而导致消息丢失。


#### Message Expiration  
正如我们已经讨论过的，Pulsar 会无限期地保留所有未确认的消息，我们防止这些消息备份的工具之一是 Backlog Quotas。 但是，它们的缺点之一是它们仅允许您根据 Topic 未确认消息所消耗的总空间来决定是否保留消息。 您还记得，Backlog Quotas 的主要原因之一是确保消费者忽略“陈旧数据”而选择更新的数据。 因此，如果有一种方法可以根据消息本身的时间准确地强制执行该操作，那将更有意义。 这就是消息过期策略发挥作用的地方。
​

Pulsar 支持 Namespace 级别的生存时间 (TTL) 策略，如果消息在称为生存时间或 TTL 的特定时间段后仍未得到确认，则允许您自动删除消息。 消息过期在使用数据的应用程序使用更新的数据而不是完整的历史更重要的情况下很有用。 我们应用程序中的一个这样的例子是司机位置数据，当食物在路上时，我们将向客户显示这些数据。 与 5 分钟前的位置相比，客户对驾驶员的最近位置更感兴趣。 因此，早于 5 分钟的驱动程序位置信息将不再相关，应清除以允许消费者仅处理较新的数据，而不是尝试处理不再有用的消息。
​

```java
# Listing 2.5 Setting Backlog Quota and Message Expiration Policies

# 定义大小限制为 2GB 和 producer_request_hold 策略的 backlog quota
./bin/pulsar-admin namespaces set-backlog-quota E-payments/payments \
--limit 2G
--policy producer_request-hold

# 设置消息过期时间 120 seconds
./bin/pulsar-admin namespaces set-message-ttl E-payments/payments \
--messageTTL 120
```
​

命名空间可以同时拥有 Backlog Quota 和与之关联的 TTL 策略，以更好地控制存储在 Pulsar Topic 中的未确认消息的保留，如清单 2.5 所示。
​

#### Message Backlog vs Message Expiration


消息保留和消息过期解决了两个根本不同的问题。 从图 2.18 中可以看出，消息保留策略仅适用于已确认的消息，并且保留那些符合保留策略的消息。 消息过期仅适用于未确认的消息，并由 TTL 设置控制，在该时间范围内未处理和确认的任何消息都将被丢弃和不处理。
​

![image.png](https://cdn.nlark.com/yuque/0/2021/png/21680022/1631416788726-be0534b0-b8b6-492c-999c-4aef7ac50ecf.png#clientId=u4e323e79-230b-4&from=paste&id=u2b384dfa&margin=%5Bobject%20Object%5D&name=image.png&originHeight=256&originWidth=632&originalType=url&ratio=1&size=33485&status=done&style=none&taskId=u7c1b721f-5722-44c6-8120-52b30da9098)


**_Figure 2.18: The backlog quota applies to messages that have not been acknowledged by all subscriptions and is based on the time-to-live setting, while the retention policy applies to acknowledged messages and is based on the volume of data to retain.  _**


 Message retention policies can be used in conjunction with tiered storage to support infinite message retention for critical datasets that you want to retain indefinitely for backup/recovery, event sourcing, or SQL exploration.  


消息保留策略可与分层存储结合使用，以支持您希望无限期保留以用于备份/恢复、事件溯源或 SQL 探索的关键数据集的无限消息保留。
​

### Tired Storage


Pulsar 的分层存储功能允许将较旧的主题数据卸载到更具成本效益的长期存储中，从而释放 Bookies 内部的磁盘空间。 对于用户而言，使用其数据存储在 Apache BookKeeper 内或分层存储上的 Topic 没有区别。 客户端仍然以相同的方式生产和消费消息，整个过程在幕后处理。


正如我们之前所讨论的，Apache Pulsar 将 Topic 存储为一个有序的 ledgers ，这些 ledgers 分布在存储层的 Bookies 中。 因为这些 ledgers  是仅追加的，所以新消息只会写入列表中的最近分类帐。 之前的所有 ledgers  都是密封的，因此 segment  内的数据是不可变的。 由于数据是不可变的，因此可以轻松地将其复制到其他存储系统，例如云存储。
​

![image.png](https://cdn.nlark.com/yuque/0/2021/png/21680022/1631417252974-fa648769-9835-4e92-ab51-0b4724900ade.png#clientId=u4e323e79-230b-4&from=paste&id=u01ec87ae&margin=%5Bobject%20Object%5D&name=image.png&originHeight=390&originWidth=647&originalType=url&ratio=1&size=37437&status=done&style=none&taskId=ud743b25c-25cb-4fde-8199-faa7867d7bc)
**_Figure 2.19: When using tiered storage, ledgers that have been closed can be copied over to Cloud Storage and removed from the Bookies to free up space. The managed ledger entries are updated to reflect the new location of the ledgers which can still be read by the topic consumers._**
​

复制完成后，可以更新 managed ledger 信息以反映数据的新存储位置，如图 2.19 所示，并且可以删除存储在 Apache BookKeeper 中的数据的原始副本。 当 ledger  被卸载到外部存储系统时，ledger  会从最旧到最新一个一个地复制到该存储系统。
​

Apache Pulsar 目前支持多个云存储系统进行分层存储，但在本节中我将重点介绍使用 AWS。 有关如何使用其他云供应商的存储系统的更多详细信息，请参阅文档。
​

####  AWS Offiload Configuration


您需要执行的第一步是创建您将用于存储 offloaded ledgers 的 S3 存储桶，并确保您将使用的 AWS 账户具有足够的权限来读取和写入存储桶中的数据。 完成后，您将需要再次修改 Broker 配置设置，如清单 2.6 所示。


```java
# Listing 2.6 Configuring AWS Tiered Storage in Pulsar

# 将卸载驱动程序类型指定为 AWS S3
managedLedgerOffloadDriver=aws-s3

# 用于 ledger 存储的 S3 存储桶名称。
s3ManagedLedgerOffloadBucket=offload-test-aws

# 存储桶所在的 AWS 区域。
s3ManagedLedgerOffloadRegion=us-east-1
    
# 如果您希望卸载程序承担 IAM 角色以执行其工作，请使用此属性。
s3ManagedLedgerOffloadRole=<aws role arn>

# 指定担任 IAM 角色时要使用的会话名称
s3ManagedLedgerOffloadRoleSessionName=pulsar-s3-offload
```


您将需要添加 AWS 特定设置以告诉 Pulsar 在 S3 中存储 ledgers  的位置。 完成这些设置后，您可以保存文件并重新启动 Pulsar 以使更改生效。
​

#### AWA  Authetication
为了让 Pulsar 将数据卸载到 S3，它必须使用一组有效的凭证向 AWS 进行身份验证。 您可能已经注意到，Pulsar 不提供任何为 AWS 配置身份验证的方法。 相反，它依赖于 DefaultAWSCredentialsProviderChain 支持的标准机制，该机制在各种预定义位置搜索 AWS 凭证。


如何你正在 AWS 上运行你的 Broker 并且其配置文件能提供凭证，如果没有提供其他机制，Pulsar 将使用这些凭证。 或者，您可以通过环境变量提供您的凭据。 最简单的方法是编辑 conf/pulsar_env.sh 文件并通过添加清单 2.7 中所示的语句“导出”环境变量 AWS_ACCESS_KEY_ID 和 AWS_SECRET_ACCESS_KEY。
```java
# Listing 2.7 Providing AWS Credentials via Environment Variables

# Add these at the beginning of the pulsar_env.sh
export AWS_ACCESS_KEY_ID=ABC123456789
export AWS_SECRET_ACCESS_KEY=ded7db27a4558e2ea8bbf0bf37ae0e8521618f366c


# or you can set them here instead.
PULSAR_EXTRA_OPTS="${PULSAR_EXTRA_OPTS} ${PULSAR_MEM} ${PULSAR_GC}
-Daws.accessKeyId=ABC123456789
-Daws.secretKey=ded7db27a4558e2ea8bbf0bf37ae0e8521618f366c
-Dio.netty.leakDetectionLevel=disabled
-Dio.netty.recycler.maxCapacity.default=1000
-Dio.netty.recycler.linkCapacity=1024"

```
您只需要使用代码清单 2.8 中显示的两种方法之一。 这两个选项同样有效，因此您可以自行选择。 但是，这两种方法都会带来安全风险，因为如果您要运行 linux ps 命令，这些 AWS 凭证将在此过程中可见。 如果您希望避免这种情况，您可以将您的凭证存储在 AWS 凭证文件的传统位置； ~/.aws/credentials ，如代码清单 2.8 所示，可以修改为对将启动 Pulsar Broker 的用户帐户具有只读权限。
```java
# Listing 2.8 Contents of the ~/.aws/credentials file

[default]
aws_access_key_id=ABC123456789
aws_secret_access_key=ded7db27a4558e2ea8bbf0bf37ae0e8521618f366c
```
但是，这种方法确实需要您将未加密的凭据存储在磁盘上，这确实会带来一些安全风险，因此不建议将其用于生产用途。
​

#### Configuring Offload To Run Automatically
仅仅因为我们配置了 managed ledger offloader 并不意味着会发生 offloading 。 我们仍然需要定义一个 Namespace 级别的策略，以便在达到某个阈值时自动 offloaded 数据。 该阈值基于 Pulsar Topic 在 BookKeeper 存储层中存储的数据总量。
​

```shell
# Listing 2.9 Configuring Automatic Offloads to Tiered Storage

/pulsar/bin/pulsar-admin namespaces set-offload-threshold \
–size 10GB \
E-payments/payments
```
​

您可以定义如清单 2.9 所示的策略，它为 Namespace 中的所有 Topic 设置 10GB 的阈值。 一旦 Topic 达到 10GB 的存储空间，就会触发所有关闭 segments  的卸载。 将阈值设置为零将导致 Broker 尽可能积极地卸载 ledgers ，并可用于最小化 BookKeeper 上存储的 Topic 数据量。 为阈值指定一个负值可以有效地完全禁用自动卸载，并且可用于具有严格 SLA 响应时间的 Topic ，这些 Topic 无法容忍从分层存储读取数据所需的额外延迟。 
​

当您有一个 Topic 想要长期保留数据时，应该使用分层存储。 一个示例是驾驶员位置 Topic，其中包含有关驾驶员在各个时间点的位置的详细信息。 您可以使用本 Topic 中的信息来训练公司的旅行时间预测模型，这些模型用于估计位置之间的旅行时间。 我们希望将这些数据保留很长时间，以开发不同时间段（几个月）的模型，以考虑天气等外部因素
​

虽然分层存储通常与具有包含大量数据的保留策略的 Topic 结合使用，但没有这样的要求。 事实上，它可以用于任何 Topic 。
