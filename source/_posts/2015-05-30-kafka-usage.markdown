---
layout: post
title: "kafka学习笔记 - 使用与配置"
date: 2015-05-30 21:01:18 +0800
comments: true
categories: kafka
---

## 目录

* [一. 使用](#一. 使用)
* [二. 关键配置](#二. 关键配置)
* [三. Storm-kafka使用](#三. Storm-kafka使用)

本文一、二部分内容主要来自官方文档。

## <a name='一. 使用'></a>一. 使用

1. 下载代码

	[https://www.apache.org/dyn/closer.cgi?path=/kafka/0.8.2.0/kafka_2.10-0.8.2.0.tgz](https://www.apache.org/dyn/closer.cgi?path=/kafka/0.8.2.0/kafka_2.10-0.8.2.0.tgz)

		tar -xzf kafka_2.10-0.8.2.0.tgz
		cd kafka_2.10-0.8.2.0
		
2. 启动服务器

	kafka依赖zookeeper，所以需要首先安装并启动zookeeper。可以使用kafka自带的zookeeper。
	
		bin/zookeeper-server-start.sh config/zookeeper.properties
		
	然后即可启动kafka
	
		bin/kafka-server-start.sh config/server.properties
		
3. 创建topic

	消息传输需要指定topic。所以首先要创建一个topic。
	
		bin/kafka-topics.sh --create --zookeeper localhost:2181 --replication-factor 1 --partitions 1 --topic test
	
	之后，可以看到已经创建的topic.其中的replication-factor指的是复制因子，即log冗余的份数，这里的数字不能大于broker的数量。
	
		bin/kafka-topics.sh --list --zookeeper localhost:2181
		
	也可以不用手动创建topic，只需要配置broker的时候设置为auto-create topic when a non-existent topic is published to.
	
4. 发送消息

	kafka提供了一个命令行客户端，可以从一个文件或者标准输入里读取并发送到kafka集群。默认的，每一行都作为一个单独的消息。
	
		bin/kafka-console-producer.sh --broker-list localhost:9092 --topic test 
		
	在命令行输入消息并回车即可发送消息。
	
5. 启动一个消费者

	kafka也提供了一个命令行消费者，接受消息并打印到标准输出。
	
		bin/kafka-console-consumer.sh --zookeeper localhost:2181 --topic test --from-beginning
	
6. 设置多broker集群
	
	首先需要为每一个broker创建一个配置文件。
	
		cp config/server.properties config/server-1.properties 
 		cp config/server.properties config/server-2.properties
 		
 		config/server-1.properties:
    		broker.id=1
    		port=9093
    		log.dirs=/tmp/kafka-logs-1
 
		config/server-2.properties:
    		broker.id=2
    		port=9094
    		log.dirs=/tmp/kafka-logs-2	
    	
    然后启动这两个结点：
    
    	bin/kafka-server-start.sh config/server-1.properties &
    	bin/kafka-server-start.sh config/server-2.properties &

    现在一共有了三个结点，三个broker，那么这样就可以形成一个集群。
    
    创建一个复制引子为3的topic
    
    	bin/kafka-topics.sh --create --zookeeper localhost:2181 --replication-factor 3 --partitions 1 --topic my-replicated-topic
    	
    如果想查看目前这个topic的partion在broker上的分布情况
    
    	bin/kafka-topics.sh --describe --zookeeper localhost:2181 --topic my-replicated-topic
    	
## <a name='二. 关键配置'></a>二. 关键配置

### 2.1 broker

- broker.id: broker的唯一标识符，集群环境该值不可重复。
- log.dirs: 一个用逗号分隔的目录列表，可以有多个，用来为Kafka存储数据。每当需要为一个新的partition分配一个目录时，会选择当前的存储partition最少的目录来存储。 
- zookeeper.connect：zookeeper访问地址，多个地址用’,’隔开
- message.max.bytes: server能接收的一条消息的最大的大小。这个属性跟consumer使用的最大fetch大小是一致的，这很重要，否则一个不守规矩的producer会发送一个太大的消息。默认值：1000000。

### 2.2 producer

- metadata.broker.list： kafka的broker列表，格式为host1:port1,host2:port2
- request.required.acks：用来控制一个produce请求怎样才能算完成，准确的说，是有多少broker必须已经提交数据到log文件，并向leader发送ack，可以设置如下的值：
	- 0，意味着producer永远不会等待一个来自broker的ack，这就是0.7版本的行为。这个选项提供了最低的延迟，但是持久化的保证是最弱的，当server挂掉的时候会丢失一些数据。
	- 1，意味着在leader replication已经接收到数据后，producer会得到一个ack。这个选项提供了更好的持久性。
	- -1，意味着在所有的ISR都接收到数据后，producer才得到一个ack。这个选项提供了最好的持久性，只要还有一个replication存活，那么数据就不会丢失。
- producer.type：决定消息是否应在一个后台线程异步发送。async表示异步发送；sync表示同步发送。设置为async则允许批量发送请求，这回带来更高的吞吐量，但是client的机器挂了的话会丢失还没有发送的数据。
- serializer.class: 消息的序列化使用的class，如kafka.serializer.StringEncoder

更多细节参见kafka.consumer.ProducerConfig类。

### 2.3 consumer

- group.id: 唯一的指明了consumer的group的名字，group名一样的进程属于同一个consumer group。
- zookeeper.connect: 通broker的配置
- consumer.id：consumer的唯一标识符，如果没有设置的话则自动生成。
- ***fetch.message.max.bytes***：每一个获取某个topic的某个partition的请求，得到最大的字节数，每一个partition的要被读取的数据会加载入内存，所以这可以帮助控制consumer使用的内存。这个值的设置不能小于在server端设置的最大消息的字节数，否则producer可能会发送大于consumer可以获取的字节数限制的消息。默认值：1024 * 1024。
- ***fetch.min.bytes***：一个fetch请求最少要返回多少字节的数据，如果数据量比这个配置少，则会等待，直到有足够的数据为止。默认值：1。
- ***fetch.wait.max.ms***：在server回应fetch请求前，如果消息不足，就是说小于fetch.min.bytes时，server最多阻塞的时间。如果超时，消息将立即发送给consumer。默认值：100。
- ***socket.receive.buffer.bytes***: socket的receiver buffer的字节大小。默认值：64 * 1024。

更多细节参见kafka.consumer.ConsumerConfig类。

## <a name='三. Storm-kafka使用'></a>三. Storm-kafka使用

Kafka很多使用场景是输出消息到Storm的，Storm本身也提供了storm-kafka的包，在使用Storm的KafkaSpout时需要注意以下几点：

- 在采用基于SimpleConsumer的消费端实现时，我们遇到过一个情况是大量的轮询导致整个环境网络的流量异常，原因是该topic一直没有新消息，consumer端的轮询没有设置等待参数，也没有在client线程里判断进行一个短暂的sleep。几乎是以死循环的方式不断跟server端通讯，尽管每次的数据包很小，但只要有几个这样的消费端足以引起网络流量的异常。这里需要设置maxWait参数，但是此参数必须与minBytes配合使用才有效。但是在storm-kafka的KafkaUtils中的fetchMessages方法中对minBytes没有设置，因此即使设置了maxWait也没有效果。这里需要自己重写KafkaUtils来解决。

		FetchRequest fetchRequest = builder.addFetch(topic, partitionId, offset, config.fetchSizeBytes).             		clientId(config.clientId).maxWait(config.fetchMaxWait).minBytes(1).build(); // 此处是修复了原来代码里没有设置minBytes
		
- 修复了上述问题后，后来还是遇到网络流量异常的情况，后来在追踪KafkaSpout源码的过程中，发现当kafka中的消息过大时，如果不设置合适的bufferSizeBytes以及fetchSizeBytes(至少要大于kafka中最大消息的大小)，那么很容易造成客户端由于bufferSizeBytes或者fetchSize设置过小，无法将消息放入buffer中也不能成功fetch而不停地去轮询服务端，从而导致网络流量异常。