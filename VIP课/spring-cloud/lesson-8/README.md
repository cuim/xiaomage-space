-----
ppt内容：
主要议题：
1、Kafka
2、Spring Kafka
3、Spring Boot Kafka
4、Spring Cloud Stream
5、Spring Cloud Stream Kafka Binder
-----
学习到内容：
1、zookeeper具有强一致性，即各个zookeeper节点要时刻保持一致，可以牺牲可用性；
2、kafka不实现任何规范，性能高灵活性好；
3、XXXTemplate一定实现 XXXOpeations接口，KafkaTemplate实现 KafkaOperations接口；
-----
# Spring Cloud Stream（上）
## Kafka
### [官方网页](http://kafka.apache.org/)
### 主要用途
* 1、消息中间件：与普通消息中间件不同的是，kafka没有实现 JMS规范；
* 2、流式计算处理；
* 3、日志；
### [下载地址](http://kafka.apache.org/downloads)
### 执行脚本目录 `...kafka_2.11-2.0.0/bin`
### 快速上手
#### 下载并且解压 kafka 压缩包
#### 以 Windows 为例，运行服务：
1. 首先打开 `cmd`，启动 `zookeeper`:
> 第一次使用，需要复制 config/zoo_sampe.cfg ，并且重命名为"zoo.cfg"
```
.\bin\zkServer.cmd
```
2. 启动 `kafka`，cmd中切换到 F:\ProgramFiles\kafka_2.11-2.0.0 路径下：
```
.\bin\windows\kafka-server-start.bat .\config\server.properties
```
#### 创建主题，主题名称为gupao：
```
./bin/windows/kafka-topics.bat --create --zookeeper localhost:2181 --replication-factor 1 --partitions 1 --topic gupao

返回：Created topic "gupao".
```
#### 生产者 往主题 gupao 发送消息，消息内容为 xiaomage：
```
bin/windows/kafka-console-producer.bat --broker-list localhost:9092 --topic gupao
>xiaomage
```
#### 消费者 从主题 gupao 接受消息：
```
bin/windows/kafka-console-consumer.bat --bootstrap-server localhost:9092 --topic gupao --from-beginning
xiaomage
```
### 同类产品比较
* ActiveMQ：实现 JMS（Java Message Service）规范，实现规范会影响性能；
* RabbitMQ：实现 AMQP（Advanced Message Queue Protocol）规范，实现规范会影响性能；
* Kafka：不实现任何规范，灵活和性能 是它的优势；
### Java中使用 Kafka标准 API
```java
package com.gupao.springcloudstreamkafka.raw.api;

import org.apache.kafka.clients.producer.KafkaProducer;
import org.apache.kafka.clients.producer.ProducerRecord;
import org.apache.kafka.clients.producer.RecordMetadata;
import org.apache.kafka.common.serialization.StringSerializer;
import java.util.Properties;
import java.util.concurrent.ExecutionException;
import java.util.concurrent.Future;

/**
 * Kafka Producer Demo(使用原始API)
 */
public class KafkaProducerDemo {

    public static void main(String[] args) throws ExecutionException, InterruptedException {

        // 初始化配置
        Properties properties = new Properties();
        properties.setProperty("bootstrap.servers", "localhost:9092");
        properties.setProperty("key.serializer", StringSerializer.class.getName());
        properties.setProperty("value.serializer", StringSerializer.class.getName());
        // 创建 Kafka Producer
        KafkaProducer<String, String> kafkaProducer = new KafkaProducer(properties);
        // 创建 Kakfa 消息  = ProducerRecord
        String topic = "gupao";
        Integer partition = 0;
        Long timestamp = System.currentTimeMillis();
        String key = "message-key";
        String value = "gupao.com";
        ProducerRecord<String, String> record =
                new ProducerRecord<String, String>(topic, partition, timestamp, key, value);
        // 发送 Kakfa 消息返回 future；future是异步执行的
        Future<RecordMetadata> metadataFuture =  kafkaProducer.send(record);
        // 强制 future执行
        metadataFuture.get();
    }
}
```
## Spring Kafka
### [官方文档](https://docs.spring.io/spring-kafka/reference/htmlsingle/)
### 设计模式
Spring 社区对 data(`spring-data`) 操作，有一个基本的模式， Template 模式：
* JDBC : `JdbcTemplate`
* Redis : `RedisTemplate`
* Kafka : `KafkaTemplate`
* JMS : `JmsTemplate`
* Rest: `RestTemplate`
>  XXXTemplate 一定实现 XXXOpeations接口；
>
> KafkaTemplate 实现了 KafkaOperations接口；
#### Maven 依赖
```xml
<dependency>
  <groupId>org.springframework.kafka</groupId>
  <artifactId>spring-kafka</artifactId>
</dependency>
```
## Spring Boot Kafka
### Maven 依赖
```
```
#### 使用`KafkaAutoConfiguration` 自动装配kafka，其中`KafkaTemplate` 会被自动装配；
```java
	@Bean
	@ConditionalOnMissingBean(KafkaTemplate.class)
	public KafkaTemplate<?, ?> kafkaTemplate(
			ProducerFactory<Object, Object> kafkaProducerFactory,
			ProducerListener<Object, Object> kafkaProducerListener) {
		KafkaTemplate<Object, Object> kafkaTemplate = new KafkaTemplate<Object, Object>(
				kafkaProducerFactory);
		kafkaTemplate.setProducerListener(kafkaProducerListener);
		kafkaTemplate.setDefaultTopic(this.properties.getTemplate().getDefaultTopic());
		return kafkaTemplate;
	}
```
#### 创建生产者
##### 增加生产者配置
`application.properties`
> 全局配置：
>
> ```properties
> ## Spring Kafka 配置信息
> spring.kafka.bootstrapServers = localhost:9092
> ```
```properties
### Kafka 生产者配置
# spring.kafka.producer.bootstrapServers = localhost:9092
spring.kafka.producer.keySerializer =org.apache.kafka.common.serialization.StringSerializer
spring.kafka.producer.valueSerializer =org.apache.kafka.common.serialization.StringSerializer
```
##### 编写发送端实现
```java
package com.gupao.springcloudstreamkafka.web.controller;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.kafka.core.KafkaTemplate;
import org.springframework.web.bind.annotation.PostMapping;
import org.springframework.web.bind.annotation.RequestParam;
import org.springframework.web.bind.annotation.RestController;

/**
 * Kafka 生产者 Controller
 */
@RestController
public class KafkaProducerController {

    private final KafkaTemplate<String, String> kafkaTemplate;

    private final String topic;

    @Autowired
    public KafkaProducerController(KafkaTemplate<String, String> kafkaTemplate,
                                   @Value("${kafka.topic}") String topic) {
        this.kafkaTemplate = kafkaTemplate;
        this.topic = topic;
    }

    @PostMapping("/message/send")
    public Boolean sendMessage(@RequestParam String message) {
        kafkaTemplate.send(topic, message);
        return true;
    }
}
```
#### 创建消费者
##### 增加消费者配置
```properties
### Kafka 消费者配置
spring.kafka.consumer.groupId = gupao-1
spring.kafka.consumer.keyDeserializer =org.apache.kafka.common.serialization.StringDeserializer
spring.kafka.consumer.valueDeserializer =org.apache.kafka.common.serialization.StringDeserializer
```
##### 编写消费端实现

```java
package com.gupao.springcloudstreamkafka.consumer;

import org.springframework.kafka.annotation.KafkaListener;
import org.springframework.stereotype.Component;

/**
 * Kafka 消费者监听器
 */
@Component
public class KafkaConsumerListener {

    @KafkaListener(topics ="${kafka.topic}")
    public void onMessage(String message) {
        System.out.println("Kafka 消费者监听器，接受到消息：" + message);
    }
}
```
## Spring Cloud Stream
## Spring Cloud Stream Binder : Kafka
## 问答部分
1. 当使用Future时，异步调用都可以使用get()方式强制执行吗？
   答：是的，get() 等待当前线程执行完毕，并且获取返回接口
2. `@KafkaListener` 和 `KafkaConsumer`有啥区别
   答：没有实质区别，主要是 编程模式。
   `@KafkaListener` 采用注解驱动
   `KafkaConsumer` 采用接口编程
3. 消费者接受消息的地方在哪？
   答：订阅并且处理后，就消失。
4. 在生产环境配置多个生产者和消费者只需要定义不同的group就可以了吗？
   答：group 是一种，要看是不是相同 Topic
5. 为了不丢失数据，消息队列的容错，和排错后的处理，如何实现的？
   答：这个依赖于 zookeeper
6. 异步接受除了打印还有什么办法处理消息吗
   答：可以处理其他逻辑，比如存储数据库
7. kafka适合什么场景下使用？
   答：高性能的 Stream 处理
8. Kafka消息一直都在，内存占用会很多吧，消息量不停产生消息咋办？
   答：Kafka 还是会删除的，并不一致一直存在
9. 怎么没看到 broker 配置？
   答：Broker 不需要设置，它是单独启动
10. consumer 为什么要分组？
    答：consumer 需要定义不同逻辑分组，以便于管理




