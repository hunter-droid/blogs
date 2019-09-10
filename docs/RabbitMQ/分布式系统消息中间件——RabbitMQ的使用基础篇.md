## 分布式系统消息中间件——RabbitMQ的使用基础篇

### 前言

&nbsp;&nbsp;&nbsp;&nbsp;我是在解决分布式事务的一致性问题时了解到RabbitMQ的，当时主要是要基于RabbitMQ来实现我们分布式系统之间对有事务可靠性要求的系统间通信的。关于分布式事务一致性问题及其常见的解决方案，可以看我另一篇博客。提到RabbitMQ，不难想到的几个关键字:消息中间件、消息队列。而消息队列不由让我想到，当时在大学学习操作系统这门课，消息队列不难想到生产者消费者模式。(PS:操作系统这门课程真的很好也很重要，其中的一些思想在我工作的很长一段一时间内给了我很大帮助和启发，给我提供了许多解决问题的思路。强烈建议每一个程序员都去学一学操作系统！)

![image](http://hunter-image.oss-cn-beijing.aliyuncs.com/18-9-8/59528487.jpg)

### 一 消息中间件

#### 1.1 简介

&nbsp;&nbsp;&nbsp;&nbsp;消息中间件也可以称消息队列，是指用高效可靠的消息传递机制进行与平台无关的数据交流，并基于数据通信来进行分布式系统的集成。通过提供消息传递和消息队列模型，可以在分布式环境下扩展进程的通信。当下主流的消息中间件有RabbitMQ、Kafka、ActiveMQ、RocketMQ等。其能在不同平台之间进行通信，常用来屏蔽各种平台协议之间的特性，实现应用程序之间的协同。其优点在于能够在客户端和服务器之间进行同步和异步的连接，并且在任何时刻都可以将消息进行传送和转发。是分布式系统中非常重要的组件，主要用来解决应用耦合、异步通信、流量削峰等问题。

#### 1.2 作用

&nbsp;&nbsp;&nbsp;&nbsp;消息中间件几大主要作用如下:

* 解耦:在项目启动之初来预测将来会碰到什么需求是极其困难的。消息中间件在处理过程中间插入了一个隐含的、基于数据的接口层，两边的处理过程都要实现这一接口，这允许你独立地扩展或修改两边的处理过程，只要确保它们遵守同样的接口约束即可。
* 冗余(存储):在某些情况下，处理数据的过程会失败。消息中间件可以把数据进行持久化直到它们已经被完全处理，通过这一方式规避了数据丢失风险。在把一个消息从消息中间件中删除之前，需要你的处理系统明确地指出该消息己经被处理完成，从而确保你的数据被安全地保存直到你使用完毕。
* 扩展性:因为消息中间件解捐了应用的处理过程，所以提高消息入队和处理的效率是很容易的，只要另外增加处理过程即可，不需要改变代码，也不需要调节参数。
* 削峰:在访问量剧增的情况下，应用仍然需要继续发挥作用，但是这样的突发流量并不常见。如果以能处理这类峰值为标准而投入资源，无疑是巨大的浪费。使用消息中间件能够使关键组件支撑突发访问压力，不会因为突发的超负荷请求而完全崩惯。
* 可恢复性:当系统一部分组件失效时，不会影响到整个系统。消息中间件降低了进程间的稿合度，所以即使一个处理消息的进程挂掉，加入消息中间件中的消息仍然可以在系统恢复后进行处理。
* 顺序保证：在大多数使用场景下，数据处理的顺序很重要，大部分消息中间件支持一定程度上的顺序性。
* 缓冲：在任何重要的系统中，都会存在需要不同处理时间的元素。消息中间件通过一个缓冲层来帮助任务最高效率地执行，写入消息中间件的处理会尽可能快速。该缓冲层有助于控制和优化数据流经过系统的速度。
* 异步通信:在很多时候应用不想也不需要立即处理消息。消息中间件提供了异步处理机制，允许应用把一些消息放入消息中间件中，但并不立即处理它，在之后需要的时候再慢慢处理。

#### 1.3 消息中间件的两种模式

##### 1.3.1 P2P模式

&nbsp;&nbsp;&nbsp;&nbsp;P2P模式包含三个角色：消息队列（Queue），发送者(Sender)，接收者(Receiver)。每个消息都被发送到一个特定的队列，接收者从队列中获取消息。队列保留着消息，直到他们被消费或超时。

P2P的特点:

* 每个消息只有一个消费者（Consumer）(即一旦被消费，消息就不再在消息队列中)
* 发送者和接收者之间在时间上没有依赖性，也就是说当发送者发送了消息之后，不管接收者有没有正在运行它不会影响到消息被发送到队列
* 接收者在成功接收消息之后需向队列应答成功 
* 如果希望发送的每个消息都会被成功处理的话，那么需要P2P模式

##### 1.3.2 Pub/Sub模式

&nbsp;&nbsp;&nbsp;&nbsp;Pub/Sub模式包含三个角色主题（Topic），发布者（Publisher），订阅者（Subscriber） 。多个发布者将消息发送到Topic,系统将这些消息传递给多个订阅者。

Pub/Sub的特点

* 每个消息可以有多个消费者
* 发布者和订阅者之间有时间上的依赖性。针对某个主题（Topic）的订阅者，它必须创建一个订阅者之后，才能消费发布者的消息。
* 为了消费消息，订阅者必须保持运行的状态。
* 如果希望发送的消息可以不被做任何处理、或者只被一个消息者处理、或者可以被多个消费者处理的话，那么可以采用Pub/Sub模型。

#### 1.4 常用中间件介绍与对比

* Kafka是LinkedIn开源的分布式发布-订阅消息系统，目前归属于Apache定级项目。Kafka主要特点是基于Pull的模式来处理消息消费，追求高吞吐量，一开始的目的就是用于日志收集和传输。0.8版本开始支持复制，不支持事务，对消息的重复、丢失、错误没有严格要求，适合产生大量数据的互联网服务的数据收集业务。

* RabbitMQ是使用Erlang语言开发的开源消息队列系统，基于AMQP协议来实现。AMQP的主要特征是面向消息、队列、路由（包括点对点和发布/订阅）、可靠性、安全。AMQP协议更多用在企业系统内，对数据一致性、稳定性和可靠性要求很高的场景，对性能和吞吐量的要求还在其次。

* RocketMQ是阿里开源的消息中间件，它是纯Java开发，具有高吞吐量、高可用性、适合大规模分布式系统应用的特点。RocketMQ思路起源于Kafka，但并不是Kafka的一个Copy，它对消息的可靠传输及事务性做了优化，目前在阿里集团被广泛应用于交易、充值、流计算、消息推送、日志流式处理、binglog分发等场景。

RabbitMQ比Kafka可靠，kafka更适合IO高吞吐的处理，一般应用在大数据日志处理或对实时性（少量延迟），可靠性（少量丢数据）要求稍低的场景使用，比如ELK日志收集。

### 二 RabbitMQ了解

#### 2.1 简介

&nbsp;&nbsp;&nbsp;&nbsp;RabbitMQ是流行的开源消息队列系统。RabbitMQ是AMQP（高级消息队列协议）的标准实现。支持多种客户端，如：Python、Ruby、.NET、Java、JMS、C、PHP、ActionScript、XMPP、STOMP等，支持AJAX，持久化。用于在分布式系统中存储转发消息，在易用性、扩展性、高可用性等方面表现不俗。是使用Erlang编写的一个开源的消息队列，本身支持很多的协议：AMQP，XMPP, SMTP, STOMP，也正是如此，使的它变的非常重量级，更适合于企业级的开发。同时实现了一个Broker构架，这意味着消息在发送给客户端时先在中心队列排队。对路由(Routing)，负载均衡(Load balance)或者数据持久化都有很好的支持。其主要特点如下:

* 可靠性
* 灵活的路由
* 扩展性
* 高可用性
* 多种协议
* 多语言客户端
* 管理界面
* 插件机制

#### 2.2 概念

&nbsp;&nbsp;&nbsp;&nbsp;RabbitMQ从整体上来看是一个典型的生产者消费者模型，主要负责接收、存储和转发消息。其整体模型架构如下图所示:![RabbitMQ 模型架构](http://hunter-image.oss-cn-beijing.aliyuncs.com/18-9-9/84394083.jpg)



我们先来看一个RabbitMQ的运转流程，稍后会对这个流程中所涉及到的一些概念进行详细的解释。

生产者:

(1)生产者连接到RabbitMQ Broker，建立一个连接( Connection)开启一个信道(Channel)
(2)生产者声明一个交换器，并设置相关属性，比如交换机类型、是否持久化等
(3)生产者声明一个队列井设置相关属性，比如是否排他、是否持久化、是否自动删除等
(4)生产者通过路由键将交换器和队列绑定起来
(5)生产者发送消息至RabbitMQ Broker，其中包含路由键、交换器等信息。
(6)相应的交换器根据接收到的路由键查找相匹配的队列。
(7)如果找到，则将从生产者发送过来的消息存入相应的队列中。
(8)如果没有找到，则根据生产者配置的属性选择丢弃还是回退给生产者
(9)关闭信道。
(10)关闭连接。'

消费者:

(1)消费者连接到RabbitMQ Broker ，建立一个连接(Connection)，开启一个信道(Channel) 。
(2)消费者向RabbitMQ Broker 请求消费相应队列中的消息，可能会设置相应的回调函数，
(3)等待RabbitMQ Broker 回应并投递相应队列中的消息，消费者接收消息。
(4)消费者确认(ack) 接收到的消息。
(5)RabbitMQ 从队列中删除相应己经被确认的消息。
(6)关闭信道。

(7)关闭连接。

##### 2.2.1 信道

这里我们主要了解两个问题：

为什么要有信道?

&nbsp;&nbsp;&nbsp;&nbsp;主要原因还是在于TCP连接的&quot;昂贵&quot;性。无论是生产者还是消费者，都需要和RabbitMQ Broker 建立连接，这个连接就是一条TCP 连接。而操作系统对于TCP连接的创建于销毁是非常昂贵的开销。假设消费者要消费消息，并根据服务需求合理调度线程，若只进行TCP连接，那么当高并发的时候，每秒可能都有成千上万的TCP连接，不仅仅是对TCP连接的浪费，也很快会超过操作系统每秒所能建立连接的数量。如果能在一条TCP连接上操作，又能保证各个线程之间的私密性就完美了，于是信道的概念出现了。

信道是什么?

&nbsp;&nbsp;&nbsp;&nbsp;信道是建立在Connection 之上的虚拟连接。当应用程序与Rabbit Broker建立TCP连接的时候，客户端紧接着可以创建一个AMQP 信道(Channel) ，每个信道都会被指派一个唯一的D。RabbitMQ 处理的每条AMQP 指令都是通过信道完成的。信道就像电缆里的光纤束。一条电缆内含有许多光纤束，允许所有的连接通过多条光线束进行传输和接收。

##### 2.2.2 生产者消费者

&nbsp;&nbsp;&nbsp;&nbsp;关于生产者消费者我们需要了解几个概念:

* Producer:生产者，即消息投递者一方。
* 消息:消息一般分两个部分:消息体(payload)和标签。标签用来描述这条消息，如:一个交换器的名称或者一个路由Key，Rabbit通过解析标签来确定消息的去向，payload是消息内容可以使一个json，数组等等。
* Consumer:消费者，就是接收消息的一方。消费者订阅RabbitMQ的队列，当消费者消费一条消息时，只是消费消息的消息体。在消息路由的过程中，会丢弃标签，存入到队列中的只有消息体。
* Broker:消息中间件的服务节点。

##### 2.2.3 队列、交换器、路由key、绑定

&nbsp;&nbsp;&nbsp;&nbsp;从RabbitMQ的运转流程我们可以知道生产者的消息是发布到交换器上的。而消费者则是从队列上获取消息的。那么消息到底是如何从交换器到队列的呢?我们先具体了解一下这几个概念。

&nbsp;&nbsp;&nbsp;&nbsp;Queue:队列，是RabbitMQ的内部对象，用于存储消息。RabbitMQ中消息只能存储在队列中。生产者投递消息到队列，消费者从队列中获取消息并消费。多个消费者可以订阅同一个队列，这时队列中的消息会被平均分摊(轮询)给多个消费者进行消费，而不是每个消费者都收到所有的消息进行消费。(注意：RabbitMQ不支持队列层面的广播消费，如果需要广播消费，可以采用一个交换器通过路由Key绑定多个队列，由多个消费者来订阅这些队列的方式。)

&nbsp;&nbsp;&nbsp;&nbsp;Exchange：交换器。在RabbitMQ中，生产者并非直接将消息投递到队列中。真实情况是，生产者将消息发送到Exchange(交换器)，由交换器将消息路由到一个或多个队列中。如果路由不到，或返回给生产者，或直接丢弃，或做其它处理。

&nbsp;&nbsp;&nbsp;&nbsp;RoutingKey:路由Key。生产者将消息发送给交换器的时候，一般会指定一个RoutingKey，用来指定这个消息的路由规则。这个路由Key需要与交换器类型和绑定键(BindingKey)联合使用才能最终生效。在交换器类型和绑定键固定的情况下，生产者可以在发送消息给交换器时通过指定RoutingKey来决定消息流向哪里。

&nbsp;&nbsp;&nbsp;&nbsp;Binding：RabbitMQ通过绑定将交换器和队列关联起来，在绑定的时候一般会指定一个绑定键，这样RabbitMQ就可以指定如何正确的路由到队列了。

&nbsp;&nbsp;&nbsp;&nbsp;从这里我们可以看到在RabbitMQ中交换器和队列实际上可以是一对多，也可以是多对多关系。交换器和队列就像我们关系数据库中的两张表。他们同归BindingKey做关联(多对多关系表)。在我们投递消息时，可以通过Exchange和RoutingKey(对应BindingKey)就可以找到相对应的队列。

&nbsp;&nbsp;&nbsp;&nbsp;RabbitMQ主要有四种类型的交换器:

* fanout：扇形交换器，它会把发送到该交换器的消息路由到所有与该交换器绑定的队列中。如果使用扇形交换器，则不会匹配路由Key。

  ![RabbitMQ Fanout交换器](http://hunter-image.oss-cn-beijing.aliyuncs.com/18-9-18/39683646.jpg)

* direct:direct交换器，会把消息路由到RoutingKey与BindingKey完全匹配的队列中。

  ![RabbitMQ direct交换器](http://hunter-image.oss-cn-beijing.aliyuncs.com/18-9-18/40498280.jpg)

* topic:完全匹配BindingKey和RoutingKey的direct交换器 有些时候并不能满足实际业务的需求。topic 类型的交换器在匹配规则上进行了扩展，它与direct 类型的交换器相似，也是将消息路由到BindingKey 和RoutingKey 相匹配的队
  列中，但这里的匹配规则有些不同，它约定:

  * RoutingKey 为一个点号&quot;.&quot;分隔的字符串(被点号&quot;.&quot;分隔开的每一段独立的字符
    串称为一个单词)λ，如&quot;hs.rabbitmq.client&quot;，&quot;com.rabbit.client&quot;等。
  * BindingKey 和RoutingKey 一样也是点号&quot;.&quot;分隔的字符串;
  * BindingKey 中可以存在两种特殊字符串&quot;\*&quot;和&quot;#&quot;，用于做模糊匹配，其中&quot;\*&quot;用于匹配一个单词，&quot;\#&quot;用于匹配多规格单词(可以是零个)。

![RabbitMQ topic交换器](http://hunter-image.oss-cn-beijing.aliyuncs.com/18-9-18/30382119.jpg)

&nbsp;&nbsp;&nbsp;&nbsp;如图:

​    · 路由键为" apple.rabbit.client" 的消息会同时路由到Queuel 和Queue2;
​    · 路由键为" orange.mq.client" 的消息只会路由到Queue2 中:
​    · 路由键为" apple.mq.demo" 的消息只会路由到Queue2 中:
​    · 路由键为" banana.rabbit.demo" 的消息只会路由到Queuel 中:
​    · 路由键为" apple.orange.banana" 的消息将会被丢弃或者返回给生产者因为它没有匹配任何路由键。

* header:headers 类型的交换器不依赖于路由键的匹配规则来路由消息，而是根据发送的消息内容中
  的headers 属性进行匹配。在绑定队列和交换器时制定一组键值对， 当发送消息到交换器时，
  RabbitMQ 会获取到该消息的headers (也是一个键值对的形式) ，对比其中的键值对是否完全
  匹配队列和交换器绑定时指定的键值对，如果完全匹配则消息会路由到该队列，否则不会路由
  到该队列。(注:该交换器类型性能较差且不实用，因此一般不会用到)。

了解了上面的概念，我们再来思考消息是如何从交换器到队列的。首先Rabbit在接收到消息时，会解析消息的标签从而得到消息的交换器与路由key信息。然后根据交换器的类型、路由key以及该交换器和队列的绑定关系来决定消息最终投递到哪个队列里面。

### 三 RabbitMQ使用

#### 3.1 RabbitMQ安装

这里我们基于docker来安装。

##### 3.1.1 拉取镜像

```
docker pull rabbitmq:management
```

##### 3.1.2 启动容器

```
docker run -d  --name rabbit -e RABBITMQ_DEFAULT_USER=admin -e RABBITMQ_DEFAULT_PASS=admin -p 15672:15672 -p 5672:5672 -p 25672:25672 -p 61613:61613 -p 1883:1883 rabbitmq:management
```

#### 3.2 RabbitMQ 客户端开发使用

这里我们以dotnet平台下RabbitMQ.Client3.6.9(可以从nuget中下载)为示例，简单介绍dotnet平台下对RabbitMQ的简单操作。更详细的内容可以从nuget中下载源码和文档进行查看。

##### 3.2.1 连接Rabbit

```c#
  ConnectionFactory factory = new ConnectionFactory();
            factory.UserName = "admin";//用户名
            factory.Password = "admin";//密码      
            factory.HostName = "192.168.17.205";//主机名
            factory.VirtualHost = "";//虚拟主机(这个暂时不需要，稍后的文章里会介绍虚拟主机的概念)
            factory.Port = 15672;//端口
            IConnection conn = factory.CreateConnection();//创建连接
```

##### 3.2.2 创建信道

```c#
    IModel channel = conn.CreateModel();
```

说明：Connection 可以用来创建多个Channel 实例，但是Channel 实例不能在线程问共享，应用程序应该为每一个线程开辟一个Channel 。某些情况下Channel 的操作可以并发运行，但是在其他情况下会导致在网络上出现错误的通信帧交错，同时也会影响友送方确认( publisherconfrrm)机制的运行，所以多线程问共享Channel实例是非线程安全的。

##### 3.2.3 交换器、队列和绑定

```C#
 channel.ExchangeDeclare("exchangeName", "direct", true);
 String queueName = channel.QueueDeclare().QueueName;
 channel.QueueBind(queueName, "exchangeName", "routingKey");
```

&nbsp;&nbsp;&nbsp;&nbsp;如上创建了一个持久化的、非自动删除的、绑定类型为direct 的交换器，同时也创建了一个非持久化的、排他的、自动删除的队列(此队列的名称由RabbitMQ 自动生成)。这里的交换器和队列也都没有设置特殊的参数。

&nbsp;&nbsp;&nbsp;&nbsp;上面的代码也展示了如何使用路由键将队列和交换器绑定起来。上面声明的队列具备如下特性: 只对当前应用中同一个Connection 层面可用，同一个Connection 的不同Channel可共用，并且也会在应用连接断开时自动删除。

&nbsp;&nbsp;&nbsp;&nbsp;上述方法根据参数不同，可以有不同的重载形式，根据自身的需要进行调用。

**ExchangeDeclare方法详解：**

ExchangeDeclare有多个重载方法，这些重载方法都是由下面这个方法中缺省的某些参数构成的。

```c#
void ExchangeDeclare(string exchange, string type, bool durable, bool autoDelete, IDictionary<string, object> arguments);
```

* exchange : 交换器的名称。
* type : 交换器的类型，常见的如fanout、direct 、topic
* durable: 设置是否持久化。durab l e 设置为true 表示持久化， 反之是非持久化。持久化可以将交换器存盘，在服务器重启的时候不会丢失相关信息。
* autoDelete : 设置是否自动删除。autoDelete 设置为true 则表示自动删除。自动删除的前提是至少有一个队列或者交换器与这个交换器绑定，之后所有与这个交换器绑定的队列或者交换器都与此解绑。注意不能错误地把这个参数理解为:"当与此交换器连接的客户端都断开时， RabbitMQ 会自动删除本交换器"。
* internal : 设置是否是内置的。如果设置为true ，则表示是内置的交换器，客户端程序无法直接发送消息到这个交换器中，只能通过交换器路由到交换器这种方式。
* argument : 其他一些结构化参数，比如alternate - exchange。

**QueueDeclare方法详解：**

QueueDeclare只有两个重载。

```c#
 QueueDeclareOk QueueDeclare();
 
 QueueDeclareOk QueueDeclare(string queue, bool durable, bool exclusive, bool autoDelete, IDictionary<string, object> arguments);
```

不带任何参数的queueDeclare 方法默认创建一个由RabbitMQ 命名的(类似这种amq.gen-LhQzlgv3GhDOv8PIDabOXA 名称，这种队列也称之为匿名队列〉、排他的、自动删除的、非持久化的队列。

* queue : 队列的名称。
* durable: 设置是否持久化。为true 则设置队列为持久化。持久化的队列会存盘，在服务器重启的时候可以保证不丢失相关信息。
* exclusive : 设置是否排他。为true 则设置队列为排他的。如果一个队列被声明为排他队列，该队列仅对首次声明它的连接可见，并在连接断开时自动删除。这里需要注意三点:排他队列是基于连接( Connection) 可见的，同一个连接的不同信道(Channel)是可以同时访问同一连接创建的排他队列; "首次"是指如果一个连接己经声明了一个排他队列，其他连接是不允许建立同名的排他队列的，这个与普通队列不同:即使该队列是持久化的，一旦连接关闭或者客户端退出，该排他队列都会被自动删除，这种队列适用于一个客户端同时发送和读取消息的应用场景。
* autoDelete: 设置是否自动删除。为true 则设置队列为自动删除。自动删除的前提是:至少有一个消费者连接到这个队列，之后所有与这个队列连接的消费者都断开时，才会自动删除。不能把这个参数错误地理解为:当连接到此队列的所有客户端断开时，这个队列自动删除"，因为生产者客户端创建这个队列，或者没有消费者客户端与这个队列连接时，都不会自动删除这个队列。
* argurnents: 设置队列的其他一些参数，如x-rnessage-ttl、x-expires、x-rnax-length、x-rnax-length-bytes、x-dead-letter-exchange、x-deadletter-routing-key, x-rnax-priority等。

**注意:生产者和消费者都能够使用queueDeclare 来声明一个队列，但是如果消费者在同一个信道上订阅了另一个队列，就无法再声明队列了。必须先取消订阅，然后将信道直为"传输"模式，之后才能声明队列。**

**QueueBind 方法详解：**

将队列和交换器绑定的方法如下：

```c#
void QueueBind(string queue, string exchange, string routingKey, IDictionary<string, object> arguments);
```

* queue: 队列名称:
* exchange: 交换器的名称:
* routingKey: 用来绑定队列和交换器的路由键;
* argument: 定义绑定的一些参数。

将队列与交换器解绑的方法如下:

```c#
QueueUnbind(string queue, string exchange, string routingKey, IDictionary<string, object> arguments);
```

其参数与绑定意义相同。

注：除队列可以绑定交换器外，交换器同样可以绑定队列。即:ExchangeBind方法，其使用方式与队列绑定相似。

##### 3.2.4 发送消息

&nbsp;&nbsp;&nbsp;&nbsp;发送消息可以使用BasicPublish方法。

```
void BasicPublish(string exchange, string routingKey, bool mandatory,IBasicProperties basicProperties, byte[] body);
```

* exchange: 交换器的名称，指明消息需要发送到哪个交换器中。如果设置为空字符串，则消息会被发送到RabbitMQ 默认的交换器中。
* routingKey : 路由键，交换器根据路由键将消息存储到相应的队列之中。
* basicProperties: 消息的基本属性集。
* body : 消息体( pay1oad ),真正需要发送的消息。
* mandatory: 是否将消息返回给生产者(会在后续的文章中介绍这个参数).

##### 3.2.5 消费消息

&nbsp;&nbsp;&nbsp;&nbsp;RabbitMQ 的消费模式分两种: 推(Push)模式和拉(Pull)模式。推模式采用BasicConsume
进行消费，而拉模式则是调用BasicGet进行消费。

**推模式:**

```c#
 EventingBasicConsumer consumer = new EventingBasicConsumer(channel);//定义消费者对象
 consumer.Received += (model, ea) =>
 {
        //do someting;
        channel.BasicAck(ea.DeliveryTag, multiple: false);//确认
 };
   channel.BasicConsume(queue: "queueName",
                        noAck: false,
                        consumer: consumer);//订阅消息
```

```c#
string BasicConsume(string queue, bool noAck, string consumerTag, bool noLocal, bool exclusive, IDictionary<string, object> arguments, IBasicConsumer consumer);
```

* queue : 队列的名称:
* noAck : 设置是否需要确认，false为需要确认。
* consumerTag: 消费者标签，用来区分多个消费者:
* noLocal : 设置为true 则表示不能将同一个Connection中生产者发送的消息传送给这个Connection中的消费者:
* exclusive : 设置是否排他
* arguments : 设置消费者的其他参数
* consumer: 指定处理消息的消费者对象。

**拉模式**

```
BasicGetResult result = channel.BasicGet("queueName", noAck: false);//获取消息

channel.BasicAck(result.DeliveryTag, multiple: false);//确认
```

##### 3.2.6 关闭连接

在应用程序使用完之后，需要关闭连接，释放资源:

```
channel.close();
conn.close() ;
```

显式地关闭Channel 是个好习惯，但这不是必须的，在Connection 关闭的时候，Channel 也会自动关闭。

### 结束语

&nbsp;&nbsp;&nbsp;&nbsp;以上简单介绍了分布式系统中消息中间件的概念与作用，以及RabbitMQ的一些基本概念与简单使用。下一篇文章将继续针对RabbitMQ进行总结。主要内容包括何时创建队列、RabbitMQ的确认机制、过期时间的使用、死信队列、以及利用RabbitMQ实现延迟队列......

### 参考

《RabbitMQ实战指南》

《RabbitMQ实战 高效部署分布式消息队列》





