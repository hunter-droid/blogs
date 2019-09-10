---
title: 分布式系统消息中间件——RabbitMQ的使用进阶篇
date: 2018-11-12 09:07:41
tags: 消息队列
categories: 消息队列
---
## 分布式系统消息中间件——RabbitMQ的使用进阶篇

### 前言

&nbsp;&nbsp;&nbsp;&nbsp;上一篇文章(https://www.cnblogs.com/hunternet/p/9668851.html)简单总结了分布式系统中的消息中间件以及RabbitMQ的基本使用，这篇文章主要总结一下RabbitMQ在日常项目开发中比较常用的几个特性。

### 一 mandatory 参数

&nbsp;&nbsp;&nbsp;&nbsp;上一篇文章中我们知道，生产者将消息发送到RabbitMQ的交换器中通过RoutingKey与BindingKey的匹配将之路由到具体的队列中以供消费者消费。那么当我们通过匹配规则找不到队列的时候，消息将何去何从呢?Rabbit给我们提供了两种方式。mandatory与备份交换器。

&nbsp;&nbsp;&nbsp;&nbsp;mandatory参数是channel.BasicPublish方法中的参数。其主要功能是消息传递过程中不可达目的地时将消息返回给生产者。当mandatory 参数设为true 时，交换器无法根据自身的类型和路由键找到一个符合条件的队列，那么RabbitMQ 会调用BasicReturn 命令将消息返回给生产者。当mandatory 参数设置为false 时。则消息直接被丢弃。其运转流程与实现代码如下(以C# RabbitMQ.Client 3.6.9为例):

![mandatory 参数](http://hunter-image.oss-cn-beijing.aliyuncs.com/18-9-20/24859650.jpg)

```c#
//连接与创建信道--后续的示例代码我们会省略掉这部分代码和释放连接
ConnectionFactory factory = new ConnectionFactory();
            factory.UserName = "admin";
            factory.Password = "admin";
            factory.HostName = "192.168.121.205";
            IConnection conn = factory.CreateConnection();//连接Rabbit
 IModel channel = conn.CreateModel();//创建信道


 channel.ExchangeDeclare("exchangeName", "direct", true);//定义交换器
 String queueName = channel.QueueDeclare("TestQueue", true, false, false, null).QueueName;//定义  队列 队列名TestQueue,持久化的,非排它的,非自动删除的。
 channel.QueueBind(queueName, "exchangeName", "routingKey");//队列绑定交换器

 var message = Encoding.UTF8.GetBytes("TestMsg");
 channel.BasicPublish("exchangeName", "routingKey", true, null, message);//发布一个可以路由到队列的消息，mandatory参数设置为true
 var message1 = Encoding.UTF8.GetBytes("TestMsg1");
 channel.BasicPublish("exchangeName", "routingKey1", true, null, message);//发布一个不可以路由到队列的消息，mandatory参数设置为true

 //生产者回调函数
 channel.BasicReturn += (model, ea) =>
 {
      //do something... 消息若不能路由到队列则会调用此回调函数。
 };

 //关闭信道与连接
 channel.close();
 conn.close() ;
```

### 二 备份交换器

&nbsp;&nbsp;&nbsp;&nbsp;当消息不能路由到队列时，通过mandatory设置参数,我们可以将消息返回给生产者处理。但这样会有一个问题，就是生产者需要开一个回调的函数来处理不能路由到的消息，这无疑会增加生产者的处理逻辑。备份交换器(Altemate Exchange)则提供了另一种方式来处理不能路由的消息。备份交换器可以将未被路由的消息存储在RabbitMQ中，在需要的时候去处理这些消息。其主要实现代码如下:

```c#
  IDictionary<string, object> args = new Dictionary<string, object>();
  args.Add("alternate-exchange", "altExchange");
  channel.ExchangeDeclare("normalExchange", "direct", true, false, args);//定义普通交换器并添加备份交换器参数
  channel.ExchangeDeclare("altExchange", "fanout", true, false, null);   //定义备份交换器，并声明为扇形交换器        
            
  channel.QueueDeclare("normalQueue", true, false, false, null);//定义普通队列
  channel.QueueBind("normalQueue", "normalExchange", "NormalRoutingKey1");//普通队列队列绑定普通交换器

  channel.QueueDeclare("altQueue", true, false, false, null);//定义备份队列
  channel.QueueBind("altQueue", "altExchange", "");//绑定备份队列与交换器

  var msg1 = Encoding.UTF8.GetBytes("TestMsg");
  channel.BasicPublish("normalExchange", "NormalRoutingKey1", false, null, msg1);//发布一个可以路由到队列的消息，消息最终会路由到normalQueue

  var msg2 = Encoding.UTF8.GetBytes("TestMsg1");
  channel.BasicPublish("normalExchange", "NormalRoutingKey2", false, null, msg2);//发布一个不可以被路由的消息，消息最终会进入altQueue
```

![备份交换器](http://hunter-image.oss-cn-beijing.aliyuncs.com/18-9-20/91801281.jpg)

备份交换器其实和普通的交换器没有太大的区别，为了方便使用，建议设置为fanout类型，若设置为direct 或者topic的类型。需要注意的是，消息被重新发送到备份交换器时的路由键和从生产者发出的路由键是一样的。考虑这样一种情况，如果备份交换器的类型是direct,并且有一个与其绑定的队列，假设绑定的路由键是key1，当某条携带路由键为key2 的消息被转发到这个备份交换器的时候，备份交换器没有匹配到合适的队列，则消息丢失。如果消息携带的路由键为keyl，则可以存储到队列中。
对于备份交换器，有以下几种特殊情况:

* 如果设置的备份交换器不存在，客户端和RabbitMQ 服务端都不会有异常出现，此时消息会丢失。
* 如果备份交换器没有绑定任何队列，客户端和RabbitMQ 服务端都不会有异常出现，此时消息会丢失。
* 如果备份交换器没有任何匹配的队列，客户端和RabbitMQ 服务端都不会有异常出现，此时消息会丢失。
* 如果备份交换器和mandatory参数一起使用，那么mandatory参数无效。

### 三 过期时间(TTL)

#### 3.1 设置消息的TTL

&nbsp;&nbsp;&nbsp;&nbsp;目前有两种方法可以设置消息的TTL。第一种方法是通过队列属性设置，队列中所有消息都有相同的过期时间。第二种方法是对消息本身进行单独设置，每条消息的TTL可以不同。如果两种方法一起使用，则消息的TTL 以两者之间较小的那个数值为准。消息在队列中的生存时间一旦超过设置的TTL值时，就会变成"死信" (Dead Message) ，消费者将无法再收到该消息。(有关死信队列请往下看)

&nbsp;&nbsp;&nbsp;&nbsp;通过队列属性设置消息TTL的方法是在channel.QueueDeclare方法中加入x-message-ttl参数实现的，这个参数的单位是毫秒。示例代码下:

```c#
IDictionary<string, object> args = new Dictionary<string, object>();
args.Add("x-message-ttl", 6000);
channel.QueueDeclare("ttlQueue", true, false, false, args);
```

&nbsp;&nbsp;&nbsp;&nbsp;如果不设置TTL.则表示此消息不会过期;如果将TTL设置为0 ，则表示除非此时可以直接将消息投递到消费者，否则该消息会被立即丢弃(或由死信队列来处理)。

&nbsp;&nbsp;&nbsp;&nbsp;针对每条消息设置TTL的方法是在channel.BasicPublish方法中加入Expiration的属性参数，单位为毫秒。关键代码如下：

```c#
 BasicProperties properties = new BasicProperties()
            {
                Expiration = "20000",//设置TTL为20000毫秒
            };
 var message = Encoding.UTF8.GetBytes("TestMsg");
 channel.BasicPublish("normalExchange", "NormalRoutingKey", true, properties, message);
```

**注意:对于第一种设置队列TTL属性的方法，一旦消息过期，就会从队列中抹去，而在第二种方法中，即使消息过期，也不会马上从队列中抹去，因为每条消息是否过期是在即将投递到消费者之前判定的。Why?在第一种方法里，队列中己过期的消息肯定在队列头部， RabbitMQ 只要定期从队头开始扫描是否有过期的消息即可。而第二种方法里，每条消息的过期时间不同，如果要删除所有过期消息势必要扫描整个队列，所以不如等到此消息即将被消费时再判定是否过期，如果过期再进行删除即可。**

#### 3.2 设置队列的TTL

&nbsp;&nbsp;&nbsp;&nbsp;注意，这里和上述通过队列设置消息的TTL不同。上面删除的是消息，而这里删除的是队列。通过channel.QueueDeclare 方法中的x-expires参数可以控制队列被自动删除前处于未使用状态的时间。这个未使用的意思是队列上没有任何的消费者，队列也没有被重新声明，并且在过期时间段内也未调用过channel.BasicGet命令。

&nbsp;&nbsp;&nbsp;&nbsp;设置队列里的TTL可以应用于类似RPC方式的回复队列，在RPC中，许多队列会被创建出来，但是却是未被使用的(有关RabbitMQ实现RPC请往下看)。RabbitMQ会确保在过期时间到达后将队列删除，但是不保障删除的动作有多及时。在RabbitMQ 重启后， 持久化的队列的过期时间会被重新计算。用于表示过期时间的x-expires参数以毫秒为单位， 井且服从和x-message-ttl一样的约束条件，不同的是它不能设置为0(会报错)。
示例代码如下：

```c#
 IDictionary<string, object> args = new Dictionary<string, object>();
 args.Add("x-expires", 6000);
 channel.QueueDeclare("ttlQueue", false, false, false, args);
```

### 四 死信队列

&nbsp;&nbsp;&nbsp;&nbsp;DLX(Dead-Letter-Exchange)死信交换器，当消息在一个队列中变成死信之后，它能被重新被发送到另一个交换器中，这个交换器就是DLX ，绑定DLX的队列就称之为死信队列。
消息变成死信主要有以下几种情况:

* 消息被拒绝(BasicReject/BasicNack) ，井且设置requeue 参数为false;(消费者确认机制将会在下一篇文章中涉及)
* 消息过期;
* 队列达到最大长度。

&nbsp;&nbsp;&nbsp;&nbsp;DLX也是一个正常的交换器，和一般的交换器没有区别，它能在任何的队列上被指定，实际上就是设置某个队列的属性。当这个队列中存在死信时，RabbitMQ 就会自动地将这个消息重新发布到设置的DLX上去，进而被路由到另一个队列，即死信队列。可以监听这个队列中的消息、以进行相应的处理。

&nbsp;&nbsp;&nbsp;&nbsp;通过在channel.QueueDeclare 方法中设置x-dead-letter-exchange参数来为这个队列添加DLX。其示例代码如下:

```c#
 channel.ExchangeDeclare("exchange.dlx", "direct", true);//定义死信交换器
 channel.ExchangeDeclare("exchange.normal", "direct", true);//定义普通交换器
 IDictionary<String, Object> args = new Dictionary<String, Object>();
 args.Add("x-message-ttl",10000);//定义消息过期时间为10000毫秒
 args.Add("x-dead-letter-exchange", "exchange.dlx");//定义exchange.dlx为死信交换器
 args.Add("x-dead-letter-routing-key", "routingkey");//定义死信交换器的绑定key,这里也可以不指定，则默认使用原队列的路由key

 channel.QueueDeclare("queue.normal", true, false, false, args);//定义普通队列
 channel.QueueBind("queue.normal", "exchange.normal", "normalKey");//普通队列交换器绑定

 channel.QueueDeclare("queue.dlx", true, false, false, null);//定义死信队列
 channel.QueueBind("queue.dlx", "exchange.dlx", "routingkey");//死信队列交换器绑定,若上方为制定死信队列路由key则这里需要使用原队列的路由key
 //发布消息
 var message = Encoding.UTF8.GetBytes("TestMsg");
 channel.BasicPublish("exchange.normal", "normalKey", null, message) ;

```

&nbsp;&nbsp;&nbsp;&nbsp;以下为死信队列的运转流程:

![死信队列](http://hunter-image.oss-cn-beijing.aliyuncs.com/18-9-20/67534593.jpg)

### 五 延迟队列

&nbsp;&nbsp;&nbsp;&nbsp;RabbitMQ本身并未提供延迟队列的功能。延迟队列是一个逻辑上的概念，可以通过过期时间+死信队列来模拟它的实现。延迟队列的逻辑架构大致如下:

![延迟队列](http://hunter-image.oss-cn-beijing.aliyuncs.com/18-9-20/79679939.jpg)

&nbsp;&nbsp;&nbsp;&nbsp;生产者将消息发送到过期时间为n的队列中，这个队列并未有消费者来消费消息，当过期时间到达时，消息会通过死信交换器被转发到死信队列中。而消费者从死信队列中消费消息。这个时候就达到了生产者发布了消息在讲过了n时间后消费者消费了消息，起到了延迟消费的作用。

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;延迟队列在我们的项目中可以应用于很多场景，如：下单后两个消息取消订单，七天自动收货，七天自动好评，密码冻结后24小时解冻，以及在分布式系统中消息补偿机制(1s后补偿,10s后补偿，5m后补偿......)。

![RabbitMQ 延迟队列应用场景](http://hunter-image.oss-cn-beijing.aliyuncs.com/18-9-20/88724997.jpg)

### 六 优先级队列

&nbsp;&nbsp;&nbsp;&nbsp;就像我们生活中的“特殊”人士一样，我们的业务上也存在一些“特殊”消息，可能需要优先进行处理，在生活上我们可能会对这部分特殊人士开辟一套VIP通道，而Rabbit同样也有这样的VIP通道(前提是在3.5的版本以后)，即优先级队列，队列中的消息会有优先级优先级高的消息具备优先被消费的特权。针对这些VIP消息，我们只需做两件事:

我们只需做两件事情：

1. 将队列声明为优先级队列，即在创建队列的时候添加参数 **x-max-priority** 以指定最大的优先级，值为0-255（整数）。
2. 为优先级消息添加优先级。

其示例代码如下:

```c#
channel.ExchangeDeclare("exchange.priority", "direct", true);//定义交换器
IDictionary<String, Object> args = new Dictionary<String, Object>();
args.Add("x-max-priority", 10);//定义优先级队列的最大优先级为10
channel.QueueDeclare("queue.priority", true, false, false, args);//定义优先级队列
channel.QueueBind("queue.priority", "exchange.priority", "priorityKey");//队列交换器绑定
BasicProperties properties = new BasicProperties()
{
    Priority =8,//设置消息优先级为8
};
var message = Encoding.UTF8.GetBytes("TestMsg8");
//发布消息
channel.BasicPublish("exchange.priority", "priorityKey", properties, message);
```

**注意：没有指定优先级的消息会将优先级以0对待。 对于超过优先级队列所定最大优先级的消息，优先级以最大优先级对待。对于相同优先级的消息，后进的排在前面。如果在消费者的消费速度大于生产者的速度且Broker 中没有消息堆积的情况下， 对发送的消息设置优先级也就没有什么实际意义。因为生产者刚发送完一条消息就被消费者消费了，那么就相当于Broker 中至多只有一条消息，对于单条消息来说优先级是没有什么意义的。**

&nbsp;&nbsp;&nbsp;&nbsp;关于优先级队列，好像违背了队列这种数据结构先进先出的原则，其具体是怎么实现的在这里就不过多讨论。有兴趣的可以自己研究研究。后续可能也会有相关的文章来分析其原理。

### 七 RPC 实现

&nbsp;&nbsp;&nbsp;&nbsp;RPC,是Remote Procedure Call 的简称，即远程过程调用。它是一种通过网络从远程计算机上请求服务，而不需要了解底层网络的技术。RPC 的主要功用是让构建分布式计算更容易，在提供强大的远程调用能力时不损失本地调用的语义简洁性。

&nbsp;&nbsp;&nbsp;&nbsp;有关RPC不多介绍，这里我们主要介绍RabbitMQ如何实现RPC。RabbitMQ 可以实现很简单的RPC。客户端发送请求消息，服务端回复响应的消息，为了接收响应的消息，我们需要在请求消息中发送一个回调队列(可以使用默认的队列)。其服务器端实现代码如下:

```c#
      static void Main(string[] args)
        {
            ConnectionFactory factory = new ConnectionFactory();
            factory.UserName = "admin";
            factory.Password = "admin";
            factory.HostName = "192.168.121.205";
            IConnection conn = factory.CreateConnection();
            IModel channel = conn.CreateModel();
            channel.QueueDeclare("RpcQueue", true, false, false, null);
            SimpleRpcServer rpc = new MySimpRpcServer(new Subscription(channel, "RpcQueue"));
            rpc.MainLoop();
        }
```

```c#
  public class MySimpRpcServer: SimpleRpcServer
    {
        public MySimpRpcServer(Subscription subscription) : base(subscription)
        {
        }

        /// <summary>
        /// 执行完成后进行回调
        /// </summary>   
        public override byte[] HandleSimpleCall(bool isRedelivered, IBasicProperties requestProperties, byte[] body, out IBasicProperties replyProperties)
        {
            replyProperties = null;
            return Encoding.UTF8.GetBytes("我收到了!");
        }
        
        /// <summary>
        /// 进行处理
        /// </summary>
        /// <param name="evt"></param>
        public override void ProcessRequest(BasicDeliverEventArgs evt)
        {
            // todo.....
            base.ProcessRequest(evt);
        }
    }
```

客户端实现代码如下:

```c#
  ConnectionFactory factory = new ConnectionFactory();
  factory.UserName = "admin";
  factory.Password = "admin";
  factory.HostName = "192.168.121.205";
  IConnection conn = factory.CreateConnection();
  IModel channel = conn.CreateModel();
   
  SimpleRpcClient client = new SimpleRpcClient(channel, "RpcQueue");
  var message = Encoding.UTF8.GetBytes("TestMsg8");
  var result = client.Call(message);
  //do somethings...
```

以上是Rabbit客户端自己帮我们封装好的Rpc客户端与服务端的逻辑。当然我们也可以自己实现，主要是借助于BasicProperties的两个参数。

* ReplyTo: 通常用来设置一个回调队列。
* CorrelationId : 用来关联请求(request) 和其调用RPC 之后的回复(response) 。

其处理流程如下:

![RabbitMQ Rpc](http://hunter-image.oss-cn-beijing.aliyuncs.com/18-9-21/77785239.jpg)

1. 当客户端启动时，创建一个匿名的回调队列。
2. 客户端为RPC 请求设置2个属性: ReplyTo用来告知RPC 服务端回复请求时的目的队列，即回调队列; Correlationld 用来标记一个请求。
3. 请求被发送到RpcQueue队列中。
4. RPC 服务端监听RpcQueue队列中的请求，当请求到来时，服务端会处理并且把带有结果的消息发送给客户端。接收的队列就是ReplyTo设定的回调队列。
5. 客户端监昕回调队列，当有消息时，检查Correlationld 属性，如果与请求匹配，那就是结果了。

### 结束语

&nbsp;&nbsp;&nbsp;&nbsp;本篇文章简单介绍了RabbitMQ在我们项目开发中常用的几种特性。这些特性可以帮助我们更好的将Rabbit用于我们不同的业务场景中。这些特性与示例，可以自己在程序中运行一下，然后通过查看Rabbit提供的web管理界面来验证其正确性(关于web管理界面不多介绍，相信大家稍微研究研究就能明白)。当然，关于Rabbit的使用，仍有许多地方在本文中没有提及，如：RabbitMQ的特色——确认机制、持久化......将在下一篇文章中再详细介绍。

