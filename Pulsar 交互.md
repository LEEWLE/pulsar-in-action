**本章涵盖**

-  在您的开发机器上运行 Pulsar 的本地实例
-  使用命令行工具管理 Pulsar 集群 
-  使用 Java、Python 和 Go 客户端库与 Pulsar 交互
-  使用 Pulsar 的命令行工具进行故障排除

​

​

现在我们已经涵盖了 Apache Pulsar 的整体架构和术语，让我们开始使用它。 对于本地开发和测试，我建议在您自己的机器上的 Docker 容器中运行 Pulsar，这提供了一种以最少的时间、精力和金钱开始使用 Pulsar 的简单方法。 对于那些更喜欢使用完整 Pulsar 集群的人，您可以参考附录 A，了解有关如何在容器化环境（如 Kubernetes）中安装和运行集群的更多详细信息。
​

在本章中，我将引导您完成使用 Java API 以编程方式发送和接收消息的过程，从使用 Pulsars 管理工具创建 Pulsar Namespace 和 Topic 的过程开始。
​

# Getting Started with Pulsar
出于本地开发和测试的目的，您可以使用 Docker 容器在开发机器上运行 Pulsar。 如果您还没有安装 Docker，您应该下载社区版并按照您的操作系统的说明进行操作。 对于那些不熟悉 Docker 的人，它是一个开源项目，用于自动将应用程序部署为可从单个命令运行的可移植、自包含映像。 每个 docker 镜像都将运行整个应用程序所需的所有独立软件组件捆绑到一个部署中。 例如，一个简单的 Web 应用程序的 Docker 映像将包括 Web 服务器、数据库和应用程序代码。 简而言之，应用程序运行所需的一切。 同样，现有的 Docker 镜像包含一个 Pulsar 代理以及必要的 Zookeeper 和 BookKeeper 组件。
​

软件开发人员可以创建这些 Docker 镜像并将它们发布到一个称为“Docker Hub”的中央存储库，为了唯一标识一个镜像，您可以在上传镜像时指定一个标签。 这使人们可以快速找到最新版本的映像并将其下载到您的开发机器上。 要启动 Pulsar Docker 容器，只需执行清单 3.1 中所示的命令，该命令将下载容器镜像并启动所有必要的组件。 请注意，我们指定了一对将在本地计算机上公开的端口（6650 和 8080）。 您将在本章后面使用这些端口与 Pulsar 集群进行交互。
​

```shell
# Listing 3.1 Running Pulsar on Desktop

# 从 DockerHub 拉取最新版本
docker pull apachepulsar/pulsar-standalone

docker run -d \
 # 为这些端口配置端口转发
 -p 6650:6650 -p 8080:8080 \
 # 将数据保留在本地驱动器上
 -v $PWD/data:/pulsar/data \
 # 指定容器名称
 --name pulsar \
# 标签
apachepulsar/pulsar-standalone
```
如果 Pulsar 已成功启动，您应该能够在 Pulsar 容器的日志文件中找到 INFO 级别的消息，表明消息服务已准备就绪，如清单 3.2 所示。 您可以通过 docker log 命令访问 Docker 日志文件，如果您的容器无法启动，您可以通过该命令找到任何问题。


```shell
# Listing 3.2 Verifying that the Pulsar cluster is

$docker logs pulsar | grep "messaging service is ready"
20:11:45.834 [main] INFO org.apache.pulsar.broker.PulsarService - messaging service is
ready
20:11:45.855 [main] INFO org.apache.pulsar.broker.PulsarService - messaging service is
ready, bootstrap service port = 8080, broker url= pulsar://localhost:6650,
cluster=standalone
```
这些日志消息表明 Pulsar Broker 已启动并正在运行并接受本地开发机器的 6650 端口上的连接。 因此，所有代码示例都将使用`pulsar://localhost:6650`  URL 从 Pulsar Broker 发送和接收数据。
​

# Administering Pulsar
Pulsar 提供了一个单一的管理层，允许您从单个端点管理整个 Pulsar 实例，包括所有子集群。 Pulsar 的管理层控制所有租户的身份验证和授权、资源隔离策略、存储配额等，如图 3.1 所示。


![image.png](https://cdn.nlark.com/yuque/0/2021/png/21680022/1631440936810-1efdd5bd-e488-4100-bd34-793e7fb6841d.png#clientId=u1e218086-92e8-4&from=paste&id=ubb4b204f&margin=%5Bobject%20Object%5D&name=image.png&originHeight=456&originWidth=541&originalType=url&ratio=1&size=49446&status=done&style=none&taskId=u465c270f-8e2d-4219-b0a1-9153d4195ce)
**_ Figure 3.1: Administrative View of Pulsar. _**
**_​_**

此管理界面允许您创建和管理 Pulsar 集群中的所有各种实体，例如租户、Namespace 和 Topic，并配置其各种安全和数据保留策略。 用户可以通过 pulsar admin 命令行界面工具或通过 Java API 以编程方式与此管理界面交互，如图 3.1 所示
​

当您启动本地独立集群时，Pulsar 会自动创建一个名为“public”租户，在该租户下会创建一个名为“default”的 Namespace。该租户空间可以用了进行开发，但这不是一个现实的生产场景，因此我将演示如何创建租户和 Namespace。
​

## Create a Tenant，Namespace，and Topic
Pulsar 在 安装目录的 bin 文件夹中提供了一个名为 pulsar-admin 的命令行界面 (CLI) 工具，在我们的例子中是在 Docker 容器内。 因此，要使用此命令行工具，必须在正在运行的 docker 容器内执行该命令。 幸运的是，Docker 通过它的 docker exec 命令提供了一种方法来做到这一点。 顾名思义，这个命令在容器内部而不是在本地机器上“执行”给定的语句。
​

您可以通过发出清单 3.3 中显示的命令序列来开始使用 pulsar-admin CLI，以创建一个名为 `persistent://manning/chapter03/example-topic` 的 Topic ，我们将在整章中使用该 Topic。
​

```shell
# Listing 3.3 Pulsar Admin Commands

# 查看实例中所有的集群
docker exec -it pulsar /pulsar/bin/pulsar-admin clusters list
"standalone"

# 查看实例中所有的租户
docker exec -it pulsar /pulsar/bin/pulsar-admin tenants list
"public"
"sample"

# 创建一个名为 manning 的租户
docker exec -it pulsar /pulsar/bin/pulsar-admin tenants create manning

# 确认租户是否创建成功
docker exec -it pulsar /pulsar/bin/pulsar-admin tenants list
"manning"
"public"
"sample"

# 在租户 manning 下创建名为 chapter03 的 Namespace
docker exec -it pulsar /pulsar/bin/pulsar-admin namespaces create manning/chapter03

# 查看租户 mainning 下的所有 Namespace
docker exec -it pulsar /pulsar/bin/pulsar-admin namespaces list manning
"manning/chapter03"

# 创建 Topic
docker exec -it pulsar /pulsar/bin/pulsar-admin topics create
persistent://manning/chapter03/example-topic

# 查看Namespace manning/chapter03 下的所有租户
docker exec -it pulsar /pulsar/bin/pulsar-admin topics list manning/chapter03
```
这些命令只是 Pulsar Admin 工具基本操作，我强烈建议您参考在线文档以了解有关 CLI 工具及其所有功能的更多详细信息。 当我们开始发布消息后需要从集群中检索一些性能指标时，我们将在本章稍后重新访问 Pulsar Admin CLI 工具。 


## Java Admin API


另一种管理 Pulsar 实例的方法是通过 Java Admin API，它提供了一个可编程接口来执行管理任务。 清单 3.4 展示了如何使用 Java API 创建 persistent://manning/chapter03/example-topic 主题。


```java
import org.apache.pulsar.client.admin.PulsarAdmin;
import org.apache.pulsar.common.policies.data.TenantInfo;
public class CreateTopic {
    public static void main(String[] args) throws Exception {
        
        # 为在运行的 Pulsar 集群创建一个管理客户端
        PulsarAdmin admin = PulsarAdmin.builder()
            .serviceHttpUrl("http://localhost:8080") #A
            .build();
        
        
        TenantInfo config = new TenantInfo(
            # 为租户指定管理员角色
            Stream.of("admin").collect(
         	Collectors.toCollection(HashSet::new)),
            # 指定租户可以操作的集群
         	Stream.of("standalone").collect(
         	Collectors.toCollection(HashSet::new)));
        
        # 创建租户
        admin.tenants().createTenant("manning", config);
        
        # 创建 Namespace
        admin.namespaces().createNamespace("manning/chapter03");
        
        # 创建 Topic
        admin.topics().createNonPartitionedTopic("persistent://manning/chapter03/example-topic"); #F
    }
}
```
此 API 提供了 CLI 工具的替代方案，当您想以编程方式创建和拆除必要的 Pulsar  Topic 而不是依赖外部工具时，它在单元测试中特别有用。
​

# Pulsar Clients
Pulsar 提供了一个名为 pulsar-client 的命令行界面 (CLI) 工具，允许您从正在运行的 Pulsar 集群中的 Topic 发送和接收消息。 此工具也位于 Pulsar 安装目录下的 bin 文件夹中，因此我们需要再次使用 docker exec 命令与此工具交互。


由于 Topic 已经创建，我们可以首先将消费者附加到它，这将建立订阅并确保没有消息丢失。 这可以通过运行清单 3.5 中所示的命令来完成。 消费者是一个“阻塞脚本”； 这意味着它将不断使用来自 Topic 的消息，直到脚本被您停止（使用 Ctrl-C）。


```java
# Listing 3.5 Starting a command-line consumer

docker exec -it pulsar /pulsar/bin/pulsar-client consume \
# 消费的 Topic 
persistent://manning/chapter03/example-topic \
# 要消费的消息数，0 表示永远消费
--num-messages 0 \
# 订阅的唯一名称
--subscription-name example-sub \
# 订阅类型为独占
--subscription-type Exclusive


INFO org.apache.pulsar.client.impl.ConnectionPool - [[id: 0xe410f77d, L:/127.0.0.1:39276 -
R:localhost/127.0.0.1:6650]] Connected to server
18:08:15.819 [pulsar-client-io-1-1] INFO
org.apache.pulsar.client.impl.ConsumerStatsRecorderImpl - Starting Pulsar consumer
perf with config: {
 # 消费者配置详情
 "topicNames" : [ ],
 "topicsPattern" : null,
 # 订阅名
 "subscriptionName" : "example-sub",
 # 订阅类型
 "subscriptionType" : "Exclusive", 
 "receiverQueueSize" : 1000,
 "acknowledgementsGroupTimeMicros" : 100000,
 "negativeAckRedeliveryDelayMicros" : 60000000,
 "maxTotalReceiverQueueSizeAcrossPartitions" : 50000,
 "consumerName" : "3d7ce",
 "ackTimeoutMillis" : 0,
 "tickDurationMillis" : 1000,
 "priorityLevel" : 0,
 "cryptoFailureAction" : "FAIL",
 "properties" : { },
 "readCompacted" : false,
 # 从最新的可用消息开始消费
 "subscriptionInitialPosition" : "Latest",
 "patternAutoDiscoveryPeriod" : 1,
 "regexSubscriptionMode" : "PersistentOnly",
 "deadLetterPolicy" : null,
 "autoUpdatePartitions" : true,
 "replicateSubscriptionState" : false,
 "resetIncludeHead" : false
}
...
18:08:15.980 [pulsar-client-io-1-1] INFO
org.apache.pulsar.client.impl.MultiTopicsConsumerImpl -
[persistent://manning/chapter02/example] [example-sub] Success subscribe new topic
persistent://manning/chapter02/example in topics consumer, partitions: 2,
allTopicPartitionsNumber: 2
18:08:47.644 [pulsar-client-io-1-1] INFO com.scurrilous.circe.checksum.Crc32cIntChecksum -
SSE4.2 CRC32C provider initialized
```
在不同的 shell 中，我们将通过发出清单 3.6 所示的命令来启动一个生产者，将包含文本“Hello Pulsar”的两条消息发送到我们刚刚启动消费者的同一 Topic。
​

```java
# Sending a message using the Pulsar command-line producer

docker exec -it pulsar /pulsar/bin/pulsar-client produce \
# 将发送消息的 Topic
persistent://manning/chapter03/example-topic \
# 发送消息的次数
--num-produce 2 \
# 消息内容
--messages "Hello Pulsar"

18:08:47.106 [pulsar-client-io-1-1] INFO org.apache.pulsar.client.impl.ConnectionPool -
[[id: 0xd47ac4ea, L:/127.0.0.1:39342 - R:localhost/127.0.0.1:6650]] Connected to
server
18:08:47.367 [pulsar-client-io-1-1] INFO
org.apache.pulsar.client.impl.ProducerStatsRecorderImpl - Starting Pulsar producer
perf with config: { 
 # producer 配置详情
 "topicName" : "persistent://manning/chapter02/example",
 "producerName" : null,
 "sendTimeoutMs" : 30000,
 "blockIfQueueFull" : false,
 "maxPendingMessages" : 1000,
 "maxPendingMessagesAcrossPartitions" : 50000,
 "messageRoutingMode" : "RoundRobinPartition",
 "hashingScheme" : "JavaStringHash",
 "cryptoFailureAction" : "FAIL",
 "batchingMaxPublishDelayMicros" : 1000,
 "batchingMaxMessages" : 1000,
 "batchingEnabled" : true,
 "compressionType" : "NONE",
 "initialSequenceId" : null,
 "autoUpdatePartitions" : true,
 "properties" : { }
}
...
# 消息发布情况
18:08:47.689 [main] INFO org.apache.pulsar.client.cli.PulsarClientTool - 2 messages
successfully produced #E
```
执行清单 3.6 中的生产者命令后，您应该在启动消费者的 shell 中看到类似于清单 3.7 的内容。 这表明消息已由生产者成功发布并被消费者接收。
```java
# Listing 3.7 Receipt of Messages in Consumer Shell
----- got message -----
key:[null], properties:[], content:Hello Pulsar
----- got message -----
key:[null], properties:[], content:Hello Pulsar

```
恭喜，您刚刚使用 Pulsar 成功发送了第一条消息！ 现在我们已经确认我们的本地 Pulsar 集群正在工作并且能够发送和接收消息，让我们看一些使用各种编程语言现实的例子。 Pulsar 提供了一个简单直观的客户端 API，它封装了来自用户的所有Broker-Client通信细节。 由于 Pulsar 的流行，该客户端有多种语言特定的实现，包括 Java、Go、Python 和 C++。 这允许我们组织中的每个团队使用他们喜欢的任何语言来实现他们的服务。
​

虽然根据您选择的编程语言，官方 Pulsar 客户端库支持的功能存在显着差异（请参阅官方客户端文档了解详细信息），但它们都支持透明的重新连接和/或连接故障转移到 Broker， 消息排队直到被 Broker 确认，以及诸如带退避的连接重试之类的启发式方法。 这允许开发人员专注于消息传递逻辑，而不必在他们的应用程序代码中处理连接异常。
​

## The Pulsar Java Client
除了我们在本章前面看到的 Java Admin API 之外，Pulsar 还提供了一个 Java 客户端，可用于创建生产者、消费者和消息 readers。 Maven 中央存储库中提供了最新版本的 Pulsar Java 客户端库。 要使用最新版本，只需将 pulsar-client 库添加到您的构建配置中，如清单 3.8 所示。


```java
# Listing 3.8 Adding the Pulsar client library to your Maven project

<!-— Inside your pom.xml -->
<properties>
     <pulsar.version>2.7.2</pulsar.version>
</properties>
<dependency>
     <groupId>org.apache.pulsar</groupId>
     <artifactId>pulsar-client</artifactId>
     <version>${pulsar.version}</version>
</dependency>
```
一旦您将 Pulsar 客户端库添加到您的项目中，您就可以通过在 Java 代码中创建客户端、生产者和消费者来开始使用它与 Pulsar 交互，我们将在下一节中看到。
​

### Pulsar Client Configuration In Java 
当应用程序想要创建生产者或消费者时，首先需要使用清单 3.9 所示的代码实例化 PulsarClient 对象，其中提供 Pulsar Broker 的 URL 以及可能需要的任何其他连接配置信息， 例如安全凭证等。
```java
# Listing 3.9 Creating a PulsarClient in Java


PulsarClient client = PulsarClient.builder()
	# Pulsar Broke 的连接 URL
	.serviceUrl("pulsar://localhost:6650")
    .build();
```
PulsarClient 对象处理创建到 Pulsar broker 的连接所涉及的所有底层细节，包括自动重试，如果 Pulsar broker 配置了 TLS，则连接实安全。 客户端实例是线程安全的，可以重用于创建和管理多个生产者和消费者
​

### Pulsar Producers In Java
在 Pulsar 中，生产者用于向 Topic 写入消息。 清单 3.10 展示了如何在创建生产者时指定要向其发送消息的 Topic 。 虽然在创建 Producer 时可以使用多种配置设置，但必须的只是 Topic 名称。
​

```java
# Listing 3.10 Creating a Pulsar Producer in Java

Producer<byte[]> producer = client.newProducer()
 .topic("persistent://manning/chapter03/example-topic")
 .create();
```
也可以将元数据附加到给定的消息，如清单 3.11 所示，它显示了如何指定用于通过Key Shared订阅进行路由的消息 Key以及一些消息属性。 此功能可用于使用有用信息标记消息，例如消息发送时间、消息发送者，例如，如果消息来自嵌入式传感器、设备 ID 等。
​

```java
# Listing 3.11 Specifying Metadata in Pulsar Messages

Producer<byte[]> producer = client.newProducer()
 .topic("persistent://manning/chapter03/example-topic")
 .create();


producer.newMessage()
    # 可以指定消息键
    .key("some-key")
    # 将消息内容作为字节数组发送
    .value("my-message".getBytes())
    # 您可以附加任意数量的属性
    .property("property_1", "value_1")
    .property("property_2", "value_2")
    .send();
```
您附加到消息的元数据值将可供消息使用者使用，然后他们可以在执行其处理逻辑时使用该信息。 例如，包含表示消息发送时间的时间戳值的属性可用于将传入消息按发生的时间顺序排序，或用于将其与来自另一个 Topic 的消息相关联。
​

### Pulsar Consumers In Java
在 Pulsar 中，消费者接口用于监听特定 Topic 并处理传入的消息。 成功处理消息后，应将确认发送回 Broker，以指示我们已完成订阅中的消息处理。 这允许 Broker 知道 Topic 中的哪条消息需要传递给订阅的下一个消费者。 在 Java 中，您可以通过指定 Topic 和订阅来创建消费者，如清单 3.12 所示。
​

```java
# Listing 3.12 Creating a Pulsar Consumer in Java
Consumer consumer = client.newConsumer()
    # 您指定要从中消费的主题
    .topic("persistent://manning/chapter03/example-topic")
    # 您必须指定订阅的唯一名称
    .subscriptionName("my-subscription")
    .subscribe();
```
subscribe 方法将尝试使用指定的订阅将消费者连接到 Topic ，如果订阅已经存在并且它不是 Shared 订阅类型之一，这可能会失败，例如，您尝试连接到已经有的独占订阅活跃的消费者。 如果您是第一次使用指定的订阅名称连接到 Topic，则会自动为您创建订阅。 每当创建新订阅时，它最初默认位于 Topic 的末尾，该订阅的消费者将开始阅读创建订阅后创建的第一条消息。 如果您连接到一个预先存在的订阅，那么它将从订阅中最早的未确认消息开始读取，如图 3.2 所示。
​

![image.png](https://cdn.nlark.com/yuque/0/2021/png/21680022/1631460487808-347db088-8559-428f-bbe7-8a092dcb77be.png#clientId=u1e218086-92e8-4&from=paste&id=ud2d02ffa&margin=%5Bobject%20Object%5D&name=image.png&originHeight=174&originWidth=499&originalType=url&ratio=1&size=17751&status=done&style=none&taskId=u209ce15c-b4c1-4645-8459-9d9771977f2)
**_Figure 3.2: The consumer starts reading messages immediate after the most recently acknowledged message in the subscription. If the subscription is new, then it starts reading the messages that are added to the topic after the subscription was created.  _**
**_​_**

一种常见的消费模式是让消费者在 while 循环内监听 Topic。 在代码清单 3.13 中，消费者持续监听消息，打印接收到的任何消息的内容，然后确认消息已被处理。 如果处理逻辑失败，我们使用否定确认在稍后的时间点重新传递消息。
​

```java
# Listing 3.13 Consuming Pulsar Messages in Java

while (true) {
     // Wait for a message
     Message msg = consumer.receive();
     try {
     	 //出来消息
         System.out.println("Message received: " +
         new String(msg.getData()));
         //确认消息
     	 consumer.acknowledge(msg);
     } catch (Exception e) {
     	 //标记消息重发
     	 consumer.negativeAcknowledge(msg);
     }
}
```
​

清单 3.13 中显示的消息使用者以同步方式处理消息，因为它用来检索消息的 receive() 方法是阻塞方法，例如，它无限期地等待新消息的到达。 虽然这对于消息量低和/或我们不关心消息发布和处理之间的延迟的某些用例可能没问题，但通常同步处理不是最好的方法。 更好的方法是以异步方式处理这些消息，如清单 3.14 所示，它依赖于 Java API 提供的 MessageListener 接口。
​

```java
# Listing 3.14 Asynchronous Message Processing in Java

package com.manning.pulsar.chapter3.consumers;
import java.util.stream.IntStream;
import org.apache.pulsar.client.api.ConsumerBuilder;
import org.apache.pulsar.client.api.PulsarClient;
import org.apache.pulsar.client.api.PulsarClientException;
import org.apache.pulsar.client.api.SubscriptionType;
public class MessageListenerExample {
public static void main() throws PulsarClientException {
    //用于连接 Pulsar 的 Pulsar 客户端。
    PulsarClient client = PulsarClient.builder()
        .serviceUrl(PULSAR_SERVICE_URL)
        .build();
    //稍后将用于创建消费者实例的消费者工厂。
    ConsumerBuilder<byte[]> consumerBuilder = client.newConsumer()
        .topic(MY_TOPIC)
        .subscriptionName(SUBSCRIPTION)
        .subscriptionType(SubscriptionType.Shared)
        .messageListener((consumer, msg) -> {
            // 收到消息时要执行的业务逻辑。
            try {
                System.out.println("Message received: " + new String(msg.getData()));
                consumer.acknowledge(msg);
            } catch (PulsarClientException e) {
            
            }
        });
    
    IntStream.range(0, 4).forEach(i -> {
        //在该主题上创建五个消费者，每个消费者都具有相同的 MessageListener 实现。
     	String name = String.format("mq-consumer-%d", i);
        try {
            // 将消费者连接到 Topic 以开始接收消息
            consumerBuilder.consumerName(name).subscribe();
        } catch (PulsarClientException e) {
            e.printStackTrace();
        }
    });
     ...
    }
}
```
当使用清单 3.14 所示的 MessageListener 接口时，您需要传入您希望在收到消息时执行的代码。 在本例中，我使用 Java Lambda 来提供内联代码，但您可以看到我仍然可以使用消费者来访问消息。 使用侦听器模式允许您将业务逻辑与线程管理分开，因为 Pulsar 使用者会自动创建一个线程池来运行 MessageListeners 实例并为您处理所有线程逻辑。 
​

综上所述，代码清单 3.15 中有一个 Java 程序，它实例化了一个 Pulsar 客户端，并使用它来创建一个生产者和一个消费者，通过“my-topic”Topic 交换消息。
```java
# Listing 3.15 Endless Pulsar Producer and Consumer Pair.
    
import org.apache.pulsar.client.api.Consumer;
import org.apache.pulsar.client.api.Message;
import org.apache.pulsar.client.api.Producer;
import org.apache.pulsar.client.api.PulsarClient;
import org.apache.pulsar.client.api.PulsarClientException;
public class BackAndForth {
    public static void main(String[] args) throws Exception {
        BackAndForth sl = new BackAndForth();
        sl.startConsumer();
        sl.startProducer();
    }
    private String serviceUrl = "pulsar://localhost:6650";

    String topic = "persistent://manning/chapter03/example-topic";;
    String subscriptionName = "my-sub";
    protected void startProducer() {
        Runnable run = () - > {
            int counter = 0;
            while (true) {
                try {
                    getProducer().newMessage()
                        .value(String.format("{id: %d, time: %tc}",
                            ++counter, new Date()).getBytes())
                        .send();
                    Thread.sleep(1000);
                } catch (final Exception ex) {}
            }
        };
        new Thread(run).start();
    }
    protected void startConsumer() {
        Runnable run = () - > {
            while (true) {
                Message < byte[] > msg = null;
                try {
                    msg = getConsumer().receive();
                    System.out.printf("Message received: %s \n",
                        new String(msg.getData()));
                    getConsumer().acknowledge(msg);
                } catch (Exception e) {
                    System.err.printf(
                        "Unable to consume message: %s \n", e.getMessage());
                    consumer.negativeAcknowledge(msg);
                }
            }
        };
        new Thread(run).start();
    }
    protected Consumer < byte[] > getConsumer() throws PulsarClientException {
        if (consumer == null) {
            consumer = getClient().newConsumer()
                .topic(topic)
                .subscriptionName(subscriptionName)
                .subscriptionType(SubscriptionType.Shared)
                .subscribe();
        }
        return consumer;
    }
    protected Producer < byte[] > getProducer() throws PulsarClientException {
        if (producer == null) {
            producer = getClient().newProducer()
                .topic(topic).create();
        }
        return producer;
    }
    protected PulsarClient getClient() throws PulsarClientException {
        if (client == null) {
            client = PulsarClient.builder()
                .serviceUrl(serviceUrl)
                .build();
        }
        return client;
    }
}
```
如您所见，此代码在同一 Topic 上创建了生产者和消费者，并在不同的线程中同时运行它们。 如果你运行这段代码，你应该会看到如代码清单 3.16 所示的输出。
```java
# Listing 3.16 Endless Pulsar Producer and Consumer Pair output

Message received: {id: 1, time: Sun Sep 06 16:24:04 PDT 2020}
Message received: {id: 2, time: Sun Sep 06 16:24:05 PDT 2020}
Message received: {id: 3, time: Sun Sep 06 16:24:06 PDT 2020}
...
```
注意我们之前发送的前两条消息是如何没有包含在输出中的，因为订阅是在这些消息发布后创建的。 这与我们稍后将 Reader 接口进行对比。
​

### Dead Letter Policy
虽然在线文档中描述了 Pulsar 消费者的很多配置选项，但我想强调的是 dead-letter-policy 配置，当您遇到无法成功处理的消息时会很有用，例如当您解析来自 Topic 的非结构化消息。 在正常处理条件下，这些消息会导致抛出异常。


在这一点上，您有几个选择：

-  第一个是捕获任何异常并简单地确认这些消息无论如何都已成功处理，这实际上是忽略了它们。 
- 另一种选择是通过否定承认它们来重新交付它们。 但是，如果消息的潜在问题无法解决，这种方法可能会导致这些消息会不断地重新传递，例如，无论您处理多少次，都无法解析的消息将始终引发异常。 
- 第三种选择是将这些有问题的消息路由到一个单独的 Topic，称为死信 Topic。 这允许您保留消息以供后续进一步处理或检查的同时避免无限重新传递循环。

​

```java
# Listing 3.17 Configure the Dead-Letter-Topic Policy on a Consumer

Consumer consumer = client.newConsumer()
    .topic("persistent://manning/chapter03/example-topic")
    .subscriptionName("my-subscription")
    .deadLetterPolicy(DeadLetterPolicy.builder()
        # 设置最大重试次数
        .maxRedeliverCount(10)
        # 设置死信 Topic 名
        .deadLetterTopic("persistent://manning/chapter03/my-dlq") 
        .subscribe();

```
要为特定消费者 dead-letter policy ，Pulsar 要求您在第一次构建时指定一些属性，例如最大重新投递次数，如代码清单 3.17 所示。 当消息超过用户指定的最大重新传递计数时，它将被发送到死信 Topic 并自动确认。 然后可以在稍后的时间点检查这些消息。
 
### Pulsar Readers In Java
reader 接口允许应用程序管理它们将消费消息的位置。 当您使用 reader  连接到 Topic 时，您必须指定 reader  在连接到 Topic 时将开始使用哪条消息。 简而言之，reader 接口为 Pulsar 客户端提供了一个低级抽象，允许他们在 Topic 中手动定位自己。


![image.png](https://cdn.nlark.com/yuque/0/2021/png/21680022/1631493183371-2ef4f25b-082a-4392-9efb-cf7665cbdbcf.png#clientId=u1e218086-92e8-4&from=paste&id=u31ef3a7a&margin=%5Bobject%20Object%5D&name=image.png&originHeight=233&originWidth=526&originalType=url&ratio=1&size=25058&status=done&style=none&taskId=u7bf41399-fa44-4892-9ea1-3c5cf3579a8)
**_Figure 3.3: When connecting to a topic, the reader interface enables you to begin with; the earliest available message, the latest available message, or an application provided message ID.  _**
**_​_**

reader 接口对于使用 Pulsar 为流处理系统提供有效的一次性处理语义等用例很有帮助。 对于此用例，流处理系统必须能够将 Topic “退回”到特定消息并在那里开始阅读。 如果您选择显式提供消息 ID，那么您的应用程序将负责提前“知道”此消息 ID，可能是从持久数据存储或缓存中获取它。 一旦你实例化了一个 PulsarClient 对象，你就可以创建一个 Reader，如代码清单 3.18 所示。
**_​_**

```java
# Listing 3.18 Creating a Pulsar Reader

Reader < byte[] > reader = client.newReader()
    # 从哪个 Topic 读取数据
    .topic("persistent://manning/chapter03/example-topic")
    .readerName("my-reader")
    # 开始消息 Id
    .startMessageId(MessageId.earliest)
    .create();
while (true) {
    Message msg = reader.readNext();
    System.out.printf("Message received: %s \n", new String(msg.getData()));
}
```
如果你运行这段代码，你应该会看到如代码清单 3.19 所示的输出，因为你将从发布到 Topic 的第一条消息开始读取，这是我们从 CLI 工具发送的两条“Hello Pulsar”消息 .
​

```java
# Listing 3.19 Earliest Message Reader Output

Message read: Hello Pulsar
Message read: Hello Pulsar
Message read: {id: 1, time: Sun Sep 06 18:11:59 PDT 2020}
Message read: {id: 2, time: Sun Sep 06 18:12:00 PDT 2020}
Message read: {id: 3, time: Sun Sep 06 18:12:01 PDT 2020}
Message read: {id: 4, time: Sun Sep 06 18:12:02 PDT 2020}
Message read: {id: 5, time: Sun Sep 06 18:12:04 PDT 2020}
Message read: {id: 6, time: Sun Sep 06 18:12:05 PDT 2020}
Message read: {id: 7, time: Sun Sep 06 18:12:06 PDT 2020}
```
在清单 3.18 所示的示例中，在指定的 Topic 上创建了一个 reader，并从 Topic 中最旧的消息开始迭代 Topic 中的每条消息。 在线文档中描述了 Pulsar reader 的几个配置选项，但在大多数情况下，默认选项就足够了。
​

# Advanced Administration
