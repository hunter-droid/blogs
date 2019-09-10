## Redis之对象篇——Redis对象系统简介

### 前言

&nbsp;&nbsp;&nbsp;&nbsp;之前几篇文章,简单介绍 Redis用到的所有主要数据结构,简单动态字符串(SDS)、双端链表、字典、压缩列表、整数集合、跳跃表。	

[图解Redis之数据结构篇——简单动态字符串SDS](<https://www.cnblogs.com/hunternet/p/9957913.html>)

[图解Redis之数据结构篇——链表](<https://www.cnblogs.com/hunternet/p/9967279.html>)

[图解Redis之数据结构篇——字典](https://www.cnblogs.com/hunternet/p/9989771.html)

[图解Redis之数据结构篇——跳跃表](https://www.cnblogs.com/hunternet/p/11248192.html)

[图解Redis之数据结构篇——整数集合](https://www.cnblogs.com/hunternet/p/11268067.html)

[图解Redis之数据结构篇——压缩列表](https://www.cnblogs.com/hunternet/p/11306690.html)

&nbsp;&nbsp;&nbsp;&nbsp;Redis并没有直接使用这些数据结构来实现键值对数据库,而是基于这些数据结构创建了一个对象系统,这个系统包含字符串对象、列表对象、哈希对象、集合对象和有序集合对象这五种类型的对象,而每种对象又通过不同的编码映射到不同的底层数据结构。

### 一、Redis对象类型和编码

&nbsp;&nbsp;&nbsp;&nbsp;Redis中的每个对象都由一个redisObject结构表示,该结构中和保存数据有关的三个属性分别是type属性、 encoding属性和ptr属性:

&nbsp;&nbsp;&nbsp;&nbsp;Redis使用对象来表示数据库中的键和值,每次当我们在Redis的数据库中新创建一个键值对时,我们至少会创建两个对象,一个对象用作键值对的健(键对象),另一个对象用作键值对的值(值对象)。

```
typedef struct redisObiect{
	//类型
	unsigned type:4;
	//编码
	unsigned encoding:4;
	//指向底层数据结构的指针
	void *ptr;
}
```

&nbsp;&nbsp;&nbsp;&nbsp;其中Redis的键对象都是字符串对象，而Redis的值对象主要有字符串、哈希、列表、集合、有序集合几种。其分别对应的内部编码和底层数据结构如下图所示：

![](https://hunter-image.oss-cn-beijing.aliyuncs.com/redis/object/Redis%E5%AF%B9%E8%B1%A1%E7%B3%BB%E7%BB%9F%E6%A8%A1%E5%9E%8B.png)



### 二、思考一个问题

&nbsp;&nbsp;&nbsp;&nbsp;Redis中的对象，大都是通过多种数据结构来实现的，为什么会这样设计呢？用一种固定的数据结构来实现，不是更加简单吗？

Redis这样设计有两个好处：

1. 可以自由改进内部编码，而对外的数据结构和命令没有影响，这样一旦开发出更优秀的内部编码，无需改动外部数据结构和命令，例如Redis3.2提供了quicklist，其结合了ziplist和linkedlist两者
   的优势，为列表类型提供了一种更为优秀的内部编码实现，而对外部用户来说基本感知不到。 这一点比较像程序设计中的分层架构。
2. 多种内部编码实现可以在不同场景下发挥各自的优势，从而优化对象在不同场景下的使用效率。例如ziplist比较节省内存，但是在列表元素比较多的情况下，性能会有所下降，这时候Redis会根据配置选项将列表类型的内部实现转换linkedlist。 (后续文章将根据具体对象介绍)

### 本文重点

* Redis基于底层的一些数据结构创建了一个对象系统以供用户使用
* 这个系统主要包含字符串对象、列表对象、哈希对象、集合对象和有序集合对象
* Redis的键对象都是字符串对象
* Redis的值对象主要有字符串、哈希、列表、集合、有序集合几种
* 为了可以自由改进内部编码，以及在不同场景下发挥其最大优势，Redis中的对象，大都是通过多种数据结构来实现


### 参考

《Redis设计与实现》

《Redis开发与运维》

《Redis官方文档》



### -----END-----

