## 分布式系统消息中间件——RabbitMQ的使用思考篇

### 前言

&nbsp;&nbsp;&nbsp;&nbsp;前面的两篇文章，我们简单介绍了消息中间件与RabbitMQ的一些基本概念、基础用法以及常用的几个特性。但如果我们想更好的去结合我们的业务场景使用好RabbitMQ，我们还需要思考一些问题。比如:何时去创建队列,RabbitMQ的持久化，如何保证消息到达RabbitMQ，以及消费者如何确认消息......

### 一、何时创建队列

&nbsp;&nbsp;&nbsp;&nbsp;从前面的文章我们知道，RabbitMQ可以选择在生产者创建队列，也可以在消费者端创建队列，也可以提前创建好队列，而生产者消费者直接使用即可。

&nbsp;&nbsp;&nbsp;&nbsp;RabbitMQ的消息存储在队列中，交换器的使用并不真正耗费服务器的性能，而队列会。如在实际业务应用中，需要对所创建的队列的流量、内存占用及网卡占用有一个清晰的认知，预估其平均值和峰值，以便在固定硬件资源的情况下能够进行合理有效的分配。

&nbsp;&nbsp;&nbsp;&nbsp;按照RabbitMQ官方建议，生产者和消费者都应该尝试创建(这里指声明操作)队列。这虽然是一个很好的建议，但是在我看来这个时间上没有最好的方案，只有最适合的方案。我们往往需要结合业务、资源等方面在各种方案里面选择一个最适合我们的方案。

&nbsp;&nbsp;&nbsp;&nbsp;如果业务本身在架构设计之初己经充分地预估了队列的使用情况，完全可以在业务程序上线之前在服务器上创建好(比如通过页面管理、RabbitMQ命令或者更好的是从配置中心下发)，这样业务程序也可以免去声明的过程，直接使用即可。预先创建好资源还有一个好处是，可以确保交换器和队列之间正确地绑定匹配。很多时候，由于人为因素、代码缺陷等，发送消息的交换器并没有绑定任何队列，那么消息将会丢失:或者交换器绑定了某个队列，但是发送消息时的路由键无法与现存的队列匹配，那么消息也会丢失。当然可以配合mandatory参数或者备份交换器(关于mandatory参数的使用详细可参考我的上一篇文章) 来提高程序的健壮性。与此同时，预估好队列的使用情况非常重要，如果在后期运行过程中超过预定的阈值，可以根据实际情况对当前集群进行扩容或者将相应的队列迁移到其他集群。迁移的过程也可以对业务程序完全透明。此种方法也更有利于开发和运维分工，便于相应资源的管理。如果集群资源充足，而即将使用的队列所占用的资源又在可控的范围之内，为了增加业务程序的灵活性，也完全可以在业务程序中声明队列。至于是使用预先分配创建资源的静态方式还是动态的创建方式，需要从业务逻辑本身、公司运维体系和公司硬件资源等方面考虑。

### 二、持久化及策略

&nbsp;&nbsp;&nbsp;&nbsp;作为一个内存中间件，在保证了速度的情况下，不可避免存在如内存数据库同样的问题，即丢失问题。持久化可以提高RabbitMQ 的可靠性，以防在异常情况(重启、关闭、宕机等)下的数据丢失。RabbitMQ的持久化分为三个部分:交换器的持久化、队列的持久化和消息的持久化。

1. 交换器的持久化

&nbsp;&nbsp;&nbsp;&nbsp;交换器的持久化是通过在声明队列是将durable 参数置为true 实现的(该参数默认为false)。如果交换器不设置持久化，那么在RabbitMQ 服务重启之后，相关的交换器元数据会丢失，不过消息不会丢失，只是不能将消息发送到这个交换器中了。对一个长期使用的交换器来说，建议将其置为持久化的。

2. 队列的持久化

&nbsp;&nbsp;&nbsp;&nbsp;队列的持久化是通过在声明队列时将durable 参数置为true 实现的(该参数默认为false)，如果队列不设置持久化，那么在RabbitMQ 服务重启之后，相关队列的元数据会丢失，此时数据也会丢失。正所谓"皮之不存，毛将焉附"，队列都没有了，消息又能存在哪里呢?

3. 消息的持久化

&nbsp;&nbsp;&nbsp;&nbsp;队列的持久化能保证其本身的元数据不会因异常情况而丢失，但是并不能保证内部所存储的消息不会丢失。要确保消息不会丢失，需要将其设置为持久化。通过将消息的投递模式(BasicProperties中的DeliveryMode属性)设置为2即可实现消息的持久化。

&nbsp;&nbsp;&nbsp;&nbsp;因此，消息如果要想在Rabbit重启、关闭、宕机时能够恢复，需要做到以下三点:

* 把消息的投递模式设置为2
* 发送到持久化的交换器
* 到达持久化的队列

&nbsp;&nbsp;&nbsp;&nbsp;**注意:RabbitMQ 确保持久化消息能从服务器重启中恢复的方式是将它们写入磁盘上的一个持久化日志文件中。当发布一条持久化消息到持久化交换器时，Rabbit会在日志提交到日志文件后才发送响应(开启生产者确认机制)。之后，如果消息到了非持久化队列，它会自动从日志文件中删除，并且无法在服务器重启后恢复。因此单单只设置队列持久化，重启之后消息会丢失;单单只设置消息的持久化，重启之后队列消失，继而消息也丢失。单单设置消息持久化而不设置队列的持久化是毫无意义的。当从持久化队列中消费了消息后(并且确认后)，RabbitMQ会在持久化日志中把这条消息标记为等待垃圾收集。而在消费持久化消息之前，若RabbitMQ服务器重启，会自动重建交换器、队列以及绑定，重播持久化日志文件中的消息到合适的队列或者交换器上(取决于宕机时，消息处在路由的哪个环节)。**

&nbsp;&nbsp;&nbsp;&nbsp;为了保障消息不会丢失，也许我们可以简单粗暴的将所有的消息标记为持久化，但这样我们会付出性能的代价。写入磁盘的速度比写入内存的速度慢得不只一点点。对于可靠性不是那么高的消息可以不采用持久化处理以提高整体的吞吐量。在选择是否要将消息持久化时，需要在可靠性和吐吞量之间做一个权衡。

&nbsp;&nbsp;&nbsp;&nbsp;将交换器、队列、消息都设置了持久化之后就能百分之百保证数据不丢失了吗?

* 从消费者来说，如果在订阅消费队列时将noAck参数设置为true ，那么当消费者接收到相关消息之后，还没来得及处理就宕机了，这样也算数据丢失。
* 在持久化的消息正确存入RabbitMQ 之后，还需要有一段时间(虽然很短，但是不可忽视〉才能存入磁盘之中。RabbitMQ 并不会为每条消息都进行同步存盘的处理，可能仅仅保存到操作系统缓存之中而不是物理磁盘之中。如果在这段时间内RabbitMQ 服务节点发生了岩机、重启等异常情况，消息保存还没来得及落盘，那么这些消息将会丢失。

&nbsp;&nbsp;&nbsp;&nbsp;关于第一个问题，可以通过消费者确认机制来解决。而第二个问题可以通过生产者确认机制来解决，也可以使用镜像队列机制(镜像队列机制，将在运维篇总结)。生产者确认消费者确认请往下看。

### 三、生产者确认

&nbsp;&nbsp;&nbsp;&nbsp;上文我们知道，在使用RabbitMQ的时候，可以通过消息持久化操作来解决因为服务器的异常崩溃而导致的消息丢失，除此之外，我们还会遇到一个问题，当消息的生产者将消息发送出去之后，消息到底有没有正确地到达服务器呢?如果不进行特殊配置，默认情况下发送消息的操作是不会返回任何信息给生产者的，也就是默认情况下生产者是不知道消息有没有正确地到达服务器。如果在消息到达服务器之前己经丢失，持久化操作也解决不了这个问题，因为消息根本没有到达服务器，何谈持久化?

&nbsp;&nbsp;&nbsp;&nbsp;RabbitMQ针对这个问题，提供了两种解决方式:

* 通过事务机制实现:
* 通过发送方确认(publisher confirm)机制实现。

#### 3.1 RabbitMQ 事务机制

&nbsp;&nbsp;&nbsp;&nbsp;RabbitMQ 客户端中与事务机制相关的方法有三个:channel.TxSelect(用于将当前信道设置为事务模式);channel.TxCommit(用于提交事务)，channel.TxRollback(用于回滚事务)。在通过channel.TxSelect方法开启事务之后，我们便可以发布消息给RabbitMQ了，如果事务提交成功，则消息一定到达了RabbitMQ 中，如果在事务提交执行之前由于RabbitMQ异常崩溃或者其他原因抛出异常，这个时候我们便可以将其捕获，进而通过执行channel.TxRollback方法来实现事务回滚。示例代码如下所示:

```c#
  channel.TxSelect();//将信道设置为事务模式
  try
  {
      //do something
      var message = Encoding.UTF8.GetBytes("TestMsg");
      channel.BasicPublish("normalExchange", "NormalRoutingKey", true, null, message);
      //do something
      channel.TxCommit();//提交事务
  }
  catch (Exception ex)
  {
      //log(ex);
      channel.TxRollback();
  }
```

&nbsp;&nbsp;&nbsp;&nbsp;事务确实能够解决消息发送方和RabbitMQ之间消息确认的问题，只有消息成功被RabbitMQ接收，事务才能提交成功，否则便可在捕获异常之后进行事务回滚，与此同时可以进行消息重发。但是使用事务同样会带来一些问题。

* 会阻塞，发布者必须等待broker处理每个消息。
* 事务是重量级的，每次提交都需要fsync()，需要耗费大量的时间
* 事务非常耗性能，会降低RabbitMQ的消息吞吐量。

#### 3.2 发送方确认机制

&nbsp;&nbsp;&nbsp;&nbsp;前面介绍了RabbitMQ可能会遇到的一个问题，即消息发送方(生产者〉并不知道消息是否真正地到达了RabbitMQ。随后了解到在AMQP协议层面提供了事务机制来解决这个问题，但是采用事务机制实现会严重降低RabbitMQ的消息吞吐量，这里就引入了一种轻量级的方式一发送方确认(publisher confirm)机制。生产者将信道设置成confirm确认)模式，一旦信道进入confirm模式，所有在该信道上面发布的消息都会被指派一个唯一的ID( 从1开始)，一旦消息被投递到所有匹配的队列之后，RabbitMQ就会发送一个确认(BasicAck) 给生产者(包含消息的唯一ID)，这就使得生产者知晓消息已经正确到达了目的地了。如果消息和队列是可持久化的，那么确认消息会在消息写入磁盘之后发出。

![RabbitMQ发送方确认机制](http://hunter-image.oss-cn-beijing.aliyuncs.com/18-11-13/17508911.jpg)

&nbsp;&nbsp;&nbsp;&nbsp;发送方确认模式，示例代码如下:

```c#
 //示例1--同步等待
 channel.ConfirmSelect();//开启确认模式
 var message = Encoding.UTF8.GetBytes("TestMsg");
 channel.ExchangeDeclare("normalExchange", "direct", true, false, null);
 channel.QueueDeclare("normalQueue", true, false, false, null);
 channel.QueueBind("normalQueue", "normalExchange", "NormalRoutingKey");
 channel.BasicPublish("normalExchange", "NormalRoutingKey", true, null, message);
 //var result=channel.WaitForConfirmsOrDie(Timeout); 
 //WaitForConfirmsOrDie 使用WaitForConfirmsOrDie 在Rabbit发送Nack命令或超时时会抛出一个异常
 var result = channel.WaitForConfirms();//等待该信道所有未确认的消息结果
 if(!result){
     //send message failed;
 }
```

```c#
 //示例2--异步通知
 channel.ConfirmSelect();//开启确认模式
 var message = Encoding.UTF8.GetBytes("TestMsg");
 channel.ExchangeDeclare("normalExchange", "direct", true, false, null);
 channel.QueueDeclare("normalQueue", true, false, false, null);
 channel.QueueBind("normalQueue", "normalExchange", "NormalRoutingKey");
 channel.BasicPublish("normalExchange", "NormalRoutingKey", true, null, message);
 channel.BasicAcks += (model, ea) =>
 {
     //消息被投递到所有匹配的队列之后，RabbitMQ就会发送一个确认(Basic.Ack)给生产者(包含消息的唯一ID)
     //ea.Multiple为True代表 ea.DeliveryTag编号之前的消息均已被确认。
	//do something;
 };
 channel.BasicNacks += (model, ea) =>
 {
     //如果RabbitMQ 因为自身内部错误导致消息丢失，就会发送一条nack(BasicNack) 命令
	//do something;
 };
```

&nbsp;&nbsp;&nbsp;&nbsp;关于生产者确认机制同样会有一些问题，broker不能保证消息会被confirm，只知道将会进行confirm。这样如果broker与生产者之间的连接断开，导致生产者不能收到确认消息，可能会重复进行发布。总之，生产者确认模式给客户端提供了一种较为轻量级的方式，能够跟踪哪些消息被broker处理，哪些可能因为broker宕掉或者网络失败的情况而重新发布。

**&nbsp;&nbsp;&nbsp;&nbsp;注意:事务机制和publisher confirm机制两者是互斥的，不能共存。如果企图将已开启事务模式的信道再设置为publisher confmn模式， RabbitMQ会报错,或者如果企图将已开启publisher confirm模式的信道设置为事务模式， RabbitMQ也会报错。在性能上来看，而到底应该选择事务机制还是Confirm机制，则需要结合我们的业务场景。**

### 四、消费者确认

&nbsp;&nbsp;&nbsp;&nbsp;为了保证消息从队列可靠地达到消费者，RabbitMQ提供了消息确认机制(message acknowledgement)。消费者在订阅队列时，可以指定noAck参数，当noAck等于false时，RabbitMQ会等待消费者显式地回复确认信号后才从内存(或者磁盘)中移去消息(实质上是先打上删除标记，之后再删除)。当noAck等于true时，RabbitMQ会自动把发送出去的消息置为确认，然后从内存(或者磁盘)中删除，而不管消费者是否真正地消费到了这些消息。

&nbsp;&nbsp;&nbsp;&nbsp;采用消息确认机制后，只要设置noAck参数为false，消费者就有足够的时间处理消息(任务)，不用担心处理消息过程中消费者进程挂掉后消息丢失的问题，因为RabbitMQ会一直等待持有消息直到消费者显式调用BasicAck命令为止。

&nbsp;&nbsp;&nbsp;&nbsp;当noAck参数置为false，对于RabbitMQ服务端而言，队列中的消息分成了两个部分:一部分是等待投递给消费者的消息:一部分是己经投递给消费者，但是还没有收到消费者确认信号的消息。如果RabbitMQ 一直没有收到消费者的确认信号，并且消费此消息的消费者己经断开连接，则RabbitMQ会安排该消息重新进入队列，等待投递给下一个消费者，当然也有可能还是原来的那个消费者。

&nbsp;&nbsp;&nbsp;&nbsp;RabbitMQ不会为未确认的消息设置过期时间，它判断此消息是否需要重新投递给消费者的唯一依据是消费该消息的消费者连接是否己经断开，这么设计的原因是RabbitMQ 允许消费者消费一条消息的时间可以很久很久。

&nbsp;&nbsp;&nbsp;&nbsp;关于RabbitMQ消费者确认机制示例代码如下:

```c#
  //推模式
  EventingBasicConsumer consumer = new EventingBasicConsumer(channel);
  //定义消费者回调事件
  consumer.Received += (model, ea) =>
  {
      //do someting;
      //channel.BasicReject(ea.DeliveryTag, requeue: true);//拒绝
      //requeue参数为true会重新将这条消息存入队列，以便可以发送给下一个订阅的消费者
      channel.BasicAck(ea.DeliveryTag, multiple: false);//确认
      //若:multiple参数为true，则确认DeliverTag这个编号之前的消息
  };
  channel.BasicConsume(queue: "queueName",
                      noAck: false,
                     consumer: consumer);

  //拉模式
  BasicGetResult result = channel.BasicGet("queueName", noAck: false);
  //确认
  channel.BasicAck(result.DeliveryTag, multiple: false);
```

![RabbitMQ 消费者确认](http://hunter-image.oss-cn-beijing.aliyuncs.com/18-9-25/36944382.jpg)

&nbsp;&nbsp;&nbsp;&nbsp;如上，消费者在消费消息的同时，Rabbit会同步给予消费者一个DeliveryTag，这个DeliveryTag就像我们数据库中的主键，消费者在消费完毕后拿着这个DeliveryTag去Rabbit确认或拒绝这个消息。

```c#
void BasicAck(ulong deliveryTag, bool multiple);

void BasicReject(ulong deliveryTag, bool requeue);

void BasicNack(ulong deliveryTag, bool multiple, bool requeue);
```

* deliveryTag:可以看作消息的编号，它是一个64位的长整型值，最大值是9223372036854775807。
* requeue:如果requeue 参数设置为true，则RabbitMQ会重新将这条消息存入队列，以便可以发送给下一个订阅的消费者;如果requeue 参数设置为false，则RabbitMQ立即会把消息从队列中移除，而不会把它发送给新的消费者。
* BasicReject命令一次只能拒绝一条消息，如果想要批量拒绝消息，则可以使用Basic.Nack这个命令。
* multiple:在BasicAck中，multiple 参数设置为true 则表示确认deliveryTag编号之前所有已被当前消费者确认的消息。在BasicNack中，multiple 参数设置为true 则表示拒绝deliveryTag 编号之前所有未被当前消费者确认的消息。

**&nbsp;&nbsp;&nbsp;&nbsp;说明:将channel.BasicReject 或者channel.BasicNack中的requeue设置为false ，可以启用"死信队列"的功能。**(关于死信队列请看我的上一篇文章 https://www.cnblogs.com/hunternet/p/9697754.html）。

&nbsp;&nbsp;&nbsp;&nbsp;上述requeue，都会将消息重新存入队列发送给下一个消费者(也有可能是其它消费者)。关于requeue还有下面一种用法。可以选择是否补发给当前的consumer。

```c#
//补发消息 true退回到queue中 /false只补发给当前的consumer
channel.BasicRecover(true);
```

**&nbsp;&nbsp;&nbsp;&nbsp;注意：RabbitMQ仅仅通过Consumer的连接中断来确认该Message并没有被正确处理。也就是说，RabbitMQ给了Consumer足够长的时间来做数据处理。如果忘记了ack，那么后果很严重。当Consumer退出时，Message会重新分发。然后RabbitMQ会占用越来越多的内存，由于RabbitMQ会长时间运行，这个“内存泄漏”是致命的。**

### 五、消息分发与顺序

#### 5.1 消息分发

&nbsp;&nbsp;&nbsp;&nbsp;当RabbitMQ 队列拥有多个消费者时，队列收到的消息将以轮询(round-robin)的分发方式发送给消费者。每条消息只会发送给订阅列表里的一个消费者。这种方式非常适合扩展，而且它是专门为并发程序设计的。如果现在负载加重，那么只需要创建更多的消费者来消费处理消息即可。
&nbsp;&nbsp;&nbsp;&nbsp;很多时候轮询的分发机制也不是那么优雅。默认情况下，如果有n个消费者，那么RabbitMQ会将第m条消息分发给第m%n (取余的方式)个消费者， RabbitMQ 不管消费者是否消费并己经确认了消息。试想一下，如果某些消费者任务繁重，来不及消费那么多的消息，而某些其他消费者由于某些原因(比如业务逻辑简单、机器性能卓越等)很快地处理完了所分配到的消息，进而进程空闲，这样就会造成整体应用吞吐量的下降。那么该如何处理这种情况呢?这里就要用到channel.BasicQos(int prefetchCount)这个方法，channel.BasicQos方法允许限制信道上的消费者所能保持的最大未确认消息的数量。
&nbsp;&nbsp;&nbsp;&nbsp;举例说明，在订阅消费队列之前，消费端程序调用了channel.BasicQos(5)，之后订阅了某个队列进行消费。RabbitMQ 会保存一个消费者的列表，每发送一条消息都会为对应的消费者计数，如果达到了所设定的上限，那么RabbitMQ 就不会向这个消费者再发送任何消息。直到消费者确认了某条消息之后， RabbitMQ 将相应的计数减1，之后消费者可以继续接收消息，直到再次到达计数上限。

**注意:Basic.Qos 的使用对于拉模式的消费方式无效.**

```c#
void BasicQos(uint prefetchSize, ushort prefetchCount, bool global);
```

* prefetchCount：允许限制信道上的消费者所能保持的最大未确认消息的数量，设置为0表示没有上限。
* prefetchSize：消费者所能接收未确认消息的总体大小的上限，单位为B，设置为0表示没有上限。
* global：对于一个信道来说，它可以同时消费多个队列，当设置了prefetchCount 大于0 时，这个信道需要和各个队列协调以确保发送的消息都没有超过所限定的prefetchCount 的值，这样会使RabbitMQ 的性能降低，尤其是这些队列分散在集群中的多个Broker节点之中。RabbitMQ 为了提升相关的性能，在AMQPO-9-1 协议之上重新定义了global这个参数。如下表所示:

| global参数 | AMQP 0-9-1                                                   | RabbitMQ                                         |
| ---------- | ------------------------------------------------------------ | ------------------------------------------------ |
| false      | 信道上所有的消费者都需要遵从prefetchCount 的限信道上新的消费者需要遵从prefetchCount 的限定值 | 信道上新的消费者需要遵从prefetchCount 的限定值   |
| true       | 当前通信链路( Connection) 上所有的消费者都需信道上所有的消费者都需要遵从prefetchCount的限定值 | 信道上所有的消费者需要遵从prefetchCount 的限定值 |

**注意:**

1. 对于同一个信道上的多个消费者而言，如果设置了prefetchCount 的值，那么都会生效。

```c#
//伪代码
Consumer consumer1 = ...;
Consumer consumer2 = ...;
channel.BasicQos(10) ; 
channel.BasicConsume("my-queue1" , false , consumer1);
channel.BasicConsume("my-queue2" , false , consumer2);
//两个消费者各自的能接收到的未确认消息的上限都为10 。
```

2. 如果在订阅消息之前，既设置了global 为true 的限制，又设置了global为false的限制,RabbitMQ 会确保两者都会生效。但会增加RabbitMQ的负载因为RabbitMQ 需要更多的资源来协调完成这些限制。

```c#
//伪代码
Channel channel = ...;
Consumer consumerl = ...;
Consumer consumer2 = ...;
channel.BasicQos(3 , false); 
channel.BasicQos(5 , true); 
channel.BasicConsume("queuel" , false , consumerl) ;
channel.BasicConsume("queue2" , false , consumer2) ;
//这里每个消费者最多只能收到3个未确认的消息，两个消费者能收到的未确认的消息个数之和的上限为5
```

#### 5.2 消息顺序

&nbsp;&nbsp;&nbsp;&nbsp;消息的顺序性是指消费者消费到的消息和发送者发布的消息的顺序是一致的。举个例子，不考虑消息重复的情况，如果生产者发布的消息分别为msgl、msg2、msg3，那么消费者必然也是按照msgl、msg2、msg3的顺序进行消费的。
&nbsp;&nbsp;&nbsp;&nbsp;目前很多资料显示RabbitMQ的消息能够保障顺序性，这是不正确的，或者说这个观点有很大的局限性。在不使用任何RabbitMQ的高级特性，也没有消息丢失、网络故障之类异常的情况发生，并且只有一个消费者的情况下，最好也只有一个生产者的情况下可以保证消息的顺序性。如果有多个生产者同时发送消息，无法确定消息到达Broker 的前后顺序，也就无法验证消息的顺序性。
&nbsp;&nbsp;&nbsp;&nbsp;那么哪些情况下RabbitMQ 的消息顺序性会被打破呢?下面介绍几种常见的情形。

* 如果生产者使用了事务机制，在发送消息之后遇到异常进行了事务回滚，那么需要重新补偿发送这条消息，如果补偿发送是在另一个线程实现的，那么消息在生产者这个源头就出现了错序。同样，如果启用publisher confirm时，在发生超时、中断，又或者是收到RabbitMQ的BasicNack命令时，那么同样需要补偿发送，结果与事务机制一样会错序。或者这种说法有些牵强，我们可以固执地认为消息的顺序性保障是从存入队列之后开始的，而不是在发迭的时候开始的。

* 考虑另一种情形，如果生产者发送的消息设置了不同的超时时间，井且也设置了死信队列，整体上来说相当于一个延迟队列，那么消费者在消费这个延迟队列的时候，消息的顺序必然不会和生产者发送消息的顺序一致。

* 如果消息设置了优先级，那么消费者消费到的消息也必然不是顺序性的。

* 如果一个队列按照前后顺序分有msg1， msg2、msg3、msg4这4 个消息，同时有ConsumerA和ConsumerB 这两个消费者同时订阅了这个队列。队列中的消息轮询分发到各个消费者之中，ConsumerA 中的消息为msg1和msg3，ConsumerB中的消息为msg2、msg4。ConsumerA收到消息msg1之后并不想处理而调用了BasicNack/BasicReject将消息拒绝，与此同时将requeue设置为true，这样这条消息就可以重新存入队列中。消息msg1之后被发送到了ConsumerB中，此时ConsumerB己经消费了msg2、msg4，之后再消费msg1.这样消息顺序性也就错乱了。

&nbsp;&nbsp;&nbsp;&nbsp;包括但不仅限于以上几种情形会使RabbitMQ 消息错序。如果要保证消息的顺序性，需要业务方使用的时候做进一步的处理。如在消息体内添加全局有序标识等。

### 六、消息传输保障

&nbsp;&nbsp;&nbsp;&nbsp;消息可靠传输一般是业务系统接入消息中间件时首要考虑的问题，一般消息中间件的消息
传输保障分为三个层级。

* At most once: 最多一次。消息可能会丢失，但绝不会重复传输。
* At least once: 最少一次。消息绝不会丢失，但可能会重复传输。
* Exactly once: 恰好一次。每条消息肯定会被传输一次且仅传输一次。

&nbsp;&nbsp;&nbsp;&nbsp;RabbitMQ 支持其中的"最多一次"和"最少一次"。其中"最少一次"投递实现需要考虑以下这个几个方面的内容:

1. 消息生产者需要开启事务机制或者publisher confirm 机制，以确保消息可以可靠地传
   输到RabbitMQ 中。
2. 消息生产者需要配合使用mandatory参数或者备份交换器来确保消息能够从交换器
   路由到队列中，进而能够保存下来而不会被丢弃。
3. 消息和队列都需要进行持久化处理，以确保RabbitMQ服务器在遇到异常情况时不会造成消息丢失。
4. 消费者在消费消息的同时需要将noAck设置为false，然后通过手动确认的方式去确认己经正确消费的消息，以避免在消费端引起不必要的消息丢失。

&nbsp;&nbsp;&nbsp;&nbsp;"最多一次"的方式就无须考虑以上那些方面，生产者随意发送，消费者随意消费，不过这样很难确保消息不会重复消费。

&nbsp;&nbsp;&nbsp;&nbsp;"恰好一次"是RabbitMQ目前无法保障的(目前我也不知道哪个中间件能够保证)。消费者在消费完一条消息之后向RabbitMQ 发送确认BasicAck命令，此时由于网络断开或者其他原因造成RabbitMQ并没有收到这个确认命令，那么RabbitMQ不会将此条消息标记删除。在重新建立连接之后，消费者还是会消费到这一条消息，这就造成了重复消费。再考虑一种情况，生产者在使用publisher confirm机制的时候，发送完一条消息等待RabbitMQ 返回确认通知，此时网络断开，生产者捕获到异常情况，为了确保消息可靠性选择重新发送，这样RabbitMQ中就有两条同样的消息，在消费的时候，消费者就会重复消费。而解决重复消费可以通过消费者幂等等方式来解决。

### 结束语

&nbsp;&nbsp;&nbsp;&nbsp;本篇文章，我们思考了使用RabbitMQ过程中需要注意的几个问题，而前两篇文章对RabbitMQ的概念以及如何使用做了简单的介绍，相信经过这些介绍已经对RabbitMQ有了基本的了解。但这些远远不够，想要更好的利用好RabbitMQ还需要结合我们的业务场景来更多的去使用它(切记不要为了使用技术而使用技术!)。关于RabbitMQ的运维篇，会在以后的文章中继续给大家分享。

