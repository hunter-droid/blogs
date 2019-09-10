## .NET下日志系统的搭建——log4net+kafka+elk

### 前言

&nbsp;&nbsp;&nbsp;&nbsp;我们公司的程序日志之前都是采用log4net记录文件日志的方式(有关log4net的简单使用可以看我另一篇博客)，但是随着后来我们团队越来越大，项目也越来越大，我们的用户量也越来越多。慢慢系统就暴露了很多问题，这个时候我们的日志系统已经不能满足我们的要求。其主要有下面几个问题:

* 随着我们访问量的增加，我们的日志文件急剧增加
* 多且乱的文件日志，难以让我们对程序进行排错
* 文件日志的记录耗用我们应用服务器的资源，导致我们的应用服务器的处理用户请求的能力下降
* 我们的日志分布在多台应用服务器上，当程序遇到问题时，我们的程序员都需要找运维人员要日志，随着团队越来越大，问题越来越多，于是导致了程序员们排队找运维要日志，解决问题的速度急剧下降！

起初，用户量不大的时候，上面的问题还能容忍。但任何一种小问题都会在用户量访问量变大的时候急剧的放大。终于在几波推广活动的时候，很悲剧的我们又不得不每天深夜加班来为我们之前对这些问题的不重视来买单。于是，在推广活动结束之后，在我们的程序员得到一丝喘息的机会时，我决定来搭建一个我们自己的日志系统，改善我们的日志记录方式。根据以上问题分析我们的日志系统需要有以下几点要求：

* 日志的写入效率要高不能对应用服务器造成太大的影响
* 要将日志集中在一台服务器上(或一组)
* 提供一个方便检索分析的可视化页面(这个最重要，再也受不了每天找运维要日志，拿到一堆文件来分析的日子了！)

一开始想要借助log4net AdoAppender把我们的日志写到数据库里，然后我们开发一个相应的功能，来对我们的日志来进行查询和分析。但考虑到写入关系数据库的性能问题，就放弃了，但有一个替代方案，就是写入到Mongo中，这样就解决了提高了一定的性能。但也需要我们开发一个功能来查询分析。这个时候从网上找了许多方案:

```
//方案1:这是我们现有的方案，优点:简单 缺点:效率低，不易查询分析，难以排错...
service-->log4net-->文件              
//方案2:优点:简单、效率高、有一定的查询分析功能 缺点:增加mongodb，增加一定复杂性，查询分析功能弱，需要投入开发精力和时间
service-->log4net-->Mongo-->开发一个功能查询分析             
//方案3:优点:性能很高，查询分析及其方便,不需要开发投入 缺点：提高了系统复杂度，需要进行大量的测试以保证其稳定性，运维需要对这些组件进行维护监控...
service-->log4net-->kafka-->logstash-->elasticsearch-->kibana搜索展示               

//其它方案
service-->log4net-->文件-->filebeat-->logstash-->elstaicsearch-->kibana

service-->log4net-->文件-->filebeat-->elstaicsearch-->kibana

service-->log4net-->文件-->logstash-->elstaicsearch-->kibana
```

最终和团队交流后决定采用方案2和方案3的结合，我增加了一个log4net for mongo的appender(这个appender,nuget上也有)，另外我们的团队开发一个能支持简单查询搜索的功能。我同步来搭建方案3。关于方案2就不多介绍了，很简单。主要提一提方案3。

### 一. ELKB简介

* Elastic Search: 从名称可以看出，Elastic Search 是用来进行搜索的，提供数据以及相应的配置信息（什么字段是什么数据类型，哪些字段可以检索等），然后你就可以自由地使用API搜索你的数据。
* Logstash：。日志文件基本上都是每行一条，每一条里面有各种信息，这个软件的功能是将每条日志解析为各个字段。
* Kibana：提供一套Web界面用来和 Elastic Search 进行交互，这样我们不用使用API来检索数据了，可以直接在 Kibana 中输入关键字，Kibana 会将返回的数据呈现给我们，当然，有很多漂亮的数据可视化图表可供选择。
* Beats：安装在每台需要收集日志的服务器上，将日志发送给Logstash进行处理，所以Beats是一个“搬运工”，将你的日志搬运到日志收集服务器上。Beats分为很多种，每一种收集特定的信息。常用的是Filebeat，监听文件变化，传送文件内容。一般日志系统使用Filebeat就够了。

### 二. kafka简介
#### 2.1 简介

kafka是一种高吞吐量的分布式发布订阅消息系统，它可以处理消费者规模的网站中的所有动作流数据。这种动作（网页浏览，搜索和其他用户的行动）是在现代网络上的许多社会功能的一个关键因素。这些数据通常是由于吞吐量的要求而通过处理日志和日志聚合来解决。

#### 2.2 适用场景

* Messaging
    对于一些常规的消息系统,kafka是个不错的选择;partitons/replication和容错,可以使kafka具有良好的扩展性和性能优势.不过到目前为止,我们应该很清楚认识到,kafka并没有提供JMS中的"事务性""消息传输担保(消息确认机制)""消息分组"等企业级特性;kafka只能使用作为"常规"的消息系统,在一定程度上,尚未确保消息的发送与接收绝对可靠(比如,消息重发,消息发送丢失等)

* Websit activity tracking
    kafka可以作为"网站活性跟踪"的最佳工具;可以将网页/用户操作等信息发送到kafka中.并实时监控,或者离线统计分析等


* Log Aggregation
    kafka的特性决定它非常适合作为"日志收集中心";application可以将操作日志"批量""异步"的发送到kafka集群中,而不是保存在本地或者DB中;kafka可以批量提交消息/压缩消息等,这对producer端而言,几乎感觉不到性能的开支.此时consumer端可以使hadoop等其他系统化的存储和分析系统.

### 三、log4net+ELK+Kafka日志系统

#### 3.1.简介

&nbsp;&nbsp;&nbsp;&nbsp;从上我们可以了解到，我们可以增加一个log4net kafkaappender 日志生产者通过这个appender将日志写入kafka，由于kafka批量提交、压缩的特性，因此对我们的应用服务器性能的开支很小。日志消费者端使用logstash订阅kafka中的消息，传送到elasticsearch中，通过kibana展示给我们。同时我们也可以通过kibana对我们的日志进行统计分析等。刚好可以解决我们上面的一些问题。整个流程大致如下图:

![image](http://hunter-image.oss-cn-beijing.aliyuncs.com/18-9-8/33635135.jpg)

关于log4net for kafka appender，我自己写了一个，nuget上也有现成的包，大家需要可以去nuget上找一找。

#### 3.2.搭建

&nbsp;&nbsp;&nbsp;&nbsp;简单介绍一下搭建，搭建过程中采用Docker。

#### 3.2.1 docker 安装kafka

```
//下载
//下载zookeeper
docker pull wurstmeister/zookeeper

//下载kafka
docker pull wurstmeister/kafka:2.11-0.11.0.3
```
```
//启动
//启动zookeeper
docker run -d --name zookeeper --publish 2181:2181 --volume /etc/localtime:/etc/localtime wurstmeister/zookeeper

//启动kafka
docker run -d --name kafka --publish 9092:9092 \
--link zookeeper \
--env KAFKA_ZOOKEEPER_CONNECT=192.168.121.205:2181 \
--env KAFKA_ADVERTISED_HOST_NAME=192.168.121.205 \
--env KAFKA_ADVERTISED_PORT=9092  \
--volume /etc/localtime:/etc/localtime \
wurstmeister/kafka:2.11-0.11.0.3
```

```
//测试
//创建topic
bin/kafka-topics.sh --create --zookeeper 192.168.121.205:2181 --replication-factor 1 --partitions 1 --topic mykafka

//查看topic
bin/kafka-topics.sh --list --zookeeper 192.168.121.205:2181

//创建生产者
bin/kafka-console-producer.sh --broker-list 192.168.121.205:9092 --topic mykafka 

//创建消费者
bin/kafka-console-consumer.sh --zookeeper 192.168.121.205:2181 --topic mykafka --from-beginning
```

#### 3.2.2 Docker安装ELK

```
//1.下载elk
docker pull sebp/elk
```
```
//2.启动elk
//Elasticsearch至少需要单独2G的内存
//增加了一个volume绑定，以免重启container以后ES的数据丢失
docker run -d -p 5044:5044 -p 127.0.0.1:5601:5601 -p 127.0.0.1:9200:9200 -p 127.0.0.1:9300:9300 -v /var/data/elk:/var/lib/elasticsearch --name=elk sebp/elk

```
```
//若启动过程出错一般是因为elasticsearch用户拥有的内存权限太小，至少需要262144
切换到root用户

执行命令：

sysctl -w vm.max_map_count=262144

查看结果：

sysctl -a|grep vm.max_map_count

显示：

vm.max_map_count = 262144
```
```
上述方法修改之后，如果重启虚拟机将失效，所以：

解决办法：

在   /etc/sysctl.conf文件最后添加一行

vm.max_map_count=262144

即可永久修改
```
启动成功之后访问：http://<your-host>:5601 看到kibana页面则说明安装成功

配置使用

```
//进入容器
docker exec -it <container-name> /bin/bash
```

```
//执行命令
/opt/logstash/bin/logstash -e 'input { stdin { } } output { elasticsearch { hosts => ["localhost"] } }'
/*
 注意：如果看到这样的报错信息 Logstash could not be started because there is already another instance using the configured data directory.  If you wish to run multiple instances, you must change the "path.data" setting. 请执行命令：service logstash stop 然后在执行就可以了。
*/
```

测试

当命令成功被执行后，看到：Successfully started Logstash API endpoint {:port=>9600} 信息后，输入：this is a dummy entry 然后回车，模拟一条日志进行测试。
打开浏览器，输入：http://<your-host>:9200/_search?pretty 如图，就会看到我们刚刚输入的日志内容。

#### 3.2.3 logstash-kafka配置实例

这是我测试用的一个配置文件。

```
input {
        kafka{
                //此处注意:logstash5.x版本以前kafka插件配置的是zookeeper地址，5.x以后配置的是kafka实例地址
                bootstrap_servers =>["192.168.121.205:9092"]
                client_id => "test" group_id => "test"
                consumer_threads => 5
                decorate_events => true
                topics => "logstash"
        }
}
filter{
        json{
                source => "message"
        }
}

output {
        elasticsearch {
                hosts => ["192.168.121.205"]
                index=> "hslog_2"
                codec => "json"
        }
}

```

配置文件启动logstash方式

```
/opt/logstash/bin/logstash -f "配置文件地址"
```

### 结束语

&nbsp;&nbsp;&nbsp;&nbsp;如上，我们的日志系统基本搭建完毕，当然还有很多关于kafka,logstash,elstaicsearch,kibana的使用，以及我们使用的一些问题，大家自己尝试着搭建一下。当然，没有最好的方案，建议大家结合自己公司和系统的现实情况，寻找和选择解决方案。能用简单的方案解决问题，就不要使用复杂的方案。因为复杂的方案在解决问题的同时，也会给我们带来其他的问题。就像我们这个方案，虽然解决了我们当时的问题，但是也增加了我们系统的复杂度，例如:这其中的每一个组件出了问题，都将导致我们的日志系统不可用......,此外，工欲善其事必先利其器，我们虽然解决了器的问题，但是要想"善我们的事"还有很长的路要走，因为究其根本，日志记不记录，在什么地方记录，记录什么等级的日志，还是由我们选择去记录。日志记录无规范、乱记、瞎记，如何规范日志的记录才是是我们接下来要解决的大问题！欢迎大家留言，探讨这些问题！