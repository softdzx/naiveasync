# NaiveAsync: 对 Kafka 生产者和消费者进行封装，提供实时报警和数据监控功能。

## Maven 依赖配置
```xml
    <dependency>
        <groupId>com.heimuheimu</groupId>
        <artifactId>naiveasync</artifactId>
        <version>1.3-SNAPSHOT</version>
    </dependency>
  
    <dependency>
        <groupId>org.apache.kafka</groupId>
        <artifactId>kafka-clients</artifactId>
        <version>0.11.0.0</version> <!-- 请选择与 Kafka 版本一致的客户端-->
    </dependency>
```

## Log4J 配置
```
log4j.logger.com.heimuheimu.naiveasync=ERROR, NAIVEASYNC
log4j.additivity.com.heimuheimu.naiveasync=false
log4j.appender.NAIVEASYNC=org.apache.log4j.DailyRollingFileAppender
log4j.appender.NAIVEASYNC.file=${log.output.directory}/naiveasync/naiveasync.log
log4j.appender.NAIVEASYNC.encoding=UTF-8
log4j.appender.NAIVEASYNC.DatePattern=_yyyy-MM-dd
log4j.appender.NAIVEASYNC.layout=org.apache.log4j.PatternLayout
log4j.appender.NAIVEASYNC.layout.ConversionPattern=%d{ISO8601} %-5p [%F:%L] : %m%n
```

## Kafka 消费者

### Spring 配置
```xml
    <!-- Kafka 消费者配置信息，更多配置项可查看 KafkaConsumerConfig 类 -->
    <bean id="kafkaConsumerConfig" class="com.heimuheimu.naiveasync.kafka.consumer.KafkaConsumerConfig">
        <property name="bootstrapServers" value="127.0.0.1:9092,127.0.0.1:9093" /> <!-- Kafka 服务地址-->
        <property name="groupId" value="consumer-group-id" /> <!-- Kafka 消费组 ID-->
    </bean>
    
    <!-- Kafka 消费者监听器，用于消费错误实时报警 -->
    <bean id="kafkaConsumerListener" class="com.heimuheimu.naiveasync.kafka.consumer.NoticeableKafkaConsumerListener">
        <constructor-arg index="0" value="your-project-name" /> <!-- 当前项目名称 -->
        <constructor-arg index="1" ref="notifierList" /> <!-- 报警器列表，关于报警器的信息可查看 naivemonitor 项目 -->
    </bean>
    
    <!-- Kafka 消费者列表，消费者为 com.heimuheimu.naiveasync.consumer.AsyncMessageConsumer<T> 的实现类 -->
    <util:list id="kafkaAsyncConsumerList">
        <bean class="com.heimuheimu.naiveasync.demo.consumer.DemoMessageConsumer" />
    </util:list>
    
    <!-- Kafka 消费者管理器 -->
    <bean id="kafkaAsyncMessageConsumerManager" class="com.heimuheimu.naiveasync.kafka.consumer.KafkaConsumerManager"
    		  init-method="init" destroy-method="close">
        <constructor-arg index="0" ref="kafkaAsyncConsumerList" />
        <constructor-arg index="1" ref="kafkaConsumerConfig" />
        <constructor-arg index="2" ref="kafkaConsumerListener" />
    </bean>
```

### Falcon 监控数据上报 Spring 配置
```xml
    <!-- 监控数据采集器列表 -->
    <util:list id="falconDataCollectorList">
        <!-- 消费者监控数据采集器 -->
        <bean class="com.heimuheimu.naiveasync.monitor.consumer.falcon.AsyncMessageConsumerDataCollector" />
        
        <!-- 如果对具体的消息类型进行额外上报，可进行以下配置
        <bean class="com.heimuheimu.naiveasync.monitor.consumer.falcon.AsyncMessageConsumerDataCollector">
            <constructor-arg index="0">
                <map>
                    <entry key="com.heimuheimu.naiveasync.demo.DemoMessage" value="demo"/>
                    <entry key="com.heimuheimu.naiveasync.demo.TestMessage" value="test"/>
                </map>
            </constructor-arg>
        </bean>
        -->
    </util:list>
    
    <!-- Falcon 监控数据上报器 -->
    <bean id="falconReporter" class="com.heimuheimu.naivemonitor.falcon.FalconReporter" init-method="init" destroy-method="close">
        <constructor-arg index="0" value="http://127.0.0.1:1988/v1/push" /> <!-- Falocn 监控数据推送地址-->
        <constructor-arg index="1" ref="falconDataCollectorList" />
    </bean>
```

### Falcon 上报数据项说明（上报周期：30秒）

* naiveasync_consumer_polled/module=naiveasync 周期内已拉取的消息总数
* naiveasync_consumer_success/module=naiveasync 周期内已消费成功的消息总数
* naiveasync_consumer_max_delay/module=naiveasync 周期内消息到达最大延迟时间（消息延迟时间 = 消息拉取时间 - 消息发送时间），单位：毫秒
* naiveasync_consumer_avg_delay/module=naiveasync 周期内消息到达平均延迟时间（消息延迟时间 = 消息拉取时间 - 消息发送时间），单位：毫秒
* naiveasync_consumer_exec_error/module=naiveasync 周期内消费出错次数，包含 Kafka 操作出现的错误和消费过程中出现的错误。

  **如果配置了具体消息类型的上报，将会有以下数据项：**

* naiveasync_consumer_{messageType}_polled/module=naiveasync 周期内该类型消息已拉取的总数
* naiveasync_consumer_{messageType}_success/module=naiveasync 周期内该类型消息已消费成功的总数
* naiveasync_consumer_{messageType}_max_delay/module=naiveasync 周期内该类型消息到达最大延迟时间，单位：毫秒
* naiveasync_consumer_{messageType}_avg_delay/module=naiveasync 周期内该类型消息到达平均延迟时间，单位：毫秒

### 消费者 Log4j 配置
```
log4j.logger.NAIVE_ASYNC_CONSUMER_INFO_LOG=INFO, NAIVE_ASYNC_CONSUMER_INFO_LOG
log4j.additivity.NAIVE_ASYNC_CONSUMER_INFO_LOG=false
log4j.appender.NAIVE_ASYNC_CONSUMER_INFO_LOG=org.apache.log4j.DailyRollingFileAppender
log4j.appender.NAIVE_ASYNC_CONSUMER_INFO_LOG.file=${log.output.directory}/naiveasync/consumer_info.log
log4j.appender.NAIVE_ASYNC_CONSUMER_INFO_LOG.encoding=UTF-8
log4j.appender.NAIVE_ASYNC_CONSUMER_INFO_LOG.DatePattern=_yyyy-MM-dd
log4j.appender.NAIVE_ASYNC_CONSUMER_INFO_LOG.layout=org.apache.log4j.PatternLayout
log4j.appender.NAIVE_ASYNC_CONSUMER_INFO_LOG.layout.ConversionPattern=%d{ISO8601} %-5p : %m%n

log4j.logger.NAIVE_ASYNC_CONSUMER_ERROR_LOG=INFO, NAIVE_ASYNC_CONSUMER_ERROR_LOG
log4j.additivity.NAIVE_ASYNC_CONSUMER_ERROR_LOG=false
log4j.appender.NAIVE_ASYNC_CONSUMER_ERROR_LOG=org.apache.log4j.DailyRollingFileAppender
log4j.appender.NAIVE_ASYNC_CONSUMER_ERROR_LOG.file=${log.output.directory}/naiveasync/consumer_error.log
log4j.appender.NAIVE_ASYNC_CONSUMER_ERROR_LOG.encoding=UTF-8
log4j.appender.NAIVE_ASYNC_CONSUMER_ERROR_LOG.DatePattern=_yyyy-MM-dd
log4j.appender.NAIVE_ASYNC_CONSUMER_ERROR_LOG.layout=org.apache.log4j.PatternLayout
log4j.appender.NAIVE_ASYNC_CONSUMER_ERROR_LOG.layout.ConversionPattern=%d{ISO8601} %-5p : %m%n
```

### 消费者示例代码

#### 单条消息消费者
```java
    public class DemoMessageConsumer extends AbstractMessageConsumer<DemoMessage> {
    
        @Override
        public void consume(DemoMessage demoMessage) {
            // 进行单条消息的消费，如果抛出异常，在等待 X 秒后，消息将会再次进行推送。
        }   
    }
```

#### 批量消息消费者
```java
    public class DemoMessageConsumer extends AbstractBatchMessageConsumer<DemoMessage> {
    
        @Override
        public void consume(List<DemoMessage> messageList) {
            // 进行批量消息的消费，如果抛出异常，在等待 X 秒后，消息列表将会再次进行推送。
        }   
    }
```

## Kafka 生产者

### Spring 配置
```xml
    <!-- Kafka 消息生产者配置信息，更多配置项可查看 KafkaProducerConfig 类 -->
    <bean id="kafkaProducerConfig" class="com.heimuheimu.naiveasync.kafka.producer.KafkaProducerConfig">
        <property name="bootstrapServers" value="127.0.0.1:9092,127.0.0.1:9093" /> <!-- Kafka 服务地址-->
    </bean>
    
    <!-- Kafka 消息生产者监听器，用于生产错误实时报警 -->
    <bean id="kafkaProducerListener" class="com.heimuheimu.naiveasync.kafka.producer.NoticeableKafkaProducerListener">
        <constructor-arg index="0" value="your-project-name" /> <!-- 当前项目名称 -->
        <constructor-arg index="1" ref="notifierList" /> <!-- 报警器列表，关于报警器的信息可查看 naivemonitor 项目 -->
    </bean>
    
    <!-- Kafka 消息生产者 -->
    <bean id="kafkaAsyncMessageProducer" class="com.heimuheimu.naiveasync.kafka.producer.KafkaProducer" destroy-method="close">
        <constructor-arg index="0" ref="kafkaProducerConfig" />
        <constructor-arg index="1" ref="kafkaProducerListener" />
    </bean>
```

### Falcon 监控数据上报 Spring 配置
```xml
    <!-- 监控数据采集器列表 -->
    <util:list id="falconDataCollectorList">
        <!-- 生产者监控数据采集器 -->
        <bean class="com.heimuheimu.naiveasync.monitor.producer.falcon.AsyncMessageProducerDataCollector" />
        
        <!-- 如果对具体的消息类型进行额外上报，可进行以下配置
        <bean class="com.heimuheimu.naiveasync.monitor.producer.falcon.AsyncMessageProducerDataCollector">
            <constructor-arg index="0">
                <map>
                    <entry key="com.heimuheimu.naiveasync.demo.DemoMessage" value="demo"/>
                    <entry key="com.heimuheimu.naiveasync.demo.TestMessage" value="test"/>
                </map>
            </constructor-arg>
        </bean>
        -->
    </util:list>
    
    <!-- Falcon 监控数据上报器 -->
    <bean id="falconReporter" class="com.heimuheimu.naivemonitor.falcon.FalconReporter" init-method="init" destroy-method="close">
        <constructor-arg index="0" value="http://127.0.0.1:1988/v1/push" /> <!-- Falocn 监控数据推送地址-->
        <constructor-arg index="1" ref="falconDataCollectorList" />
    </bean>
```

### Falcon 上报数据项说明（上报周期：30秒）

* naiveasync_producer_success/module=naiveasync 周期内发送成功的消息总数
* naiveasync_producer_error/module=naiveasync 周期内发送失败的消息总数

  **如果配置了具体消息类型的上报，将会有以下数据项：**

* naiveasync_producer_{messageType}_success/module=naiveasync 周期内该类型消息发送成功的总数
* naiveasync_producer_{messageType}_error/module=naiveasync 周期内该类型消息已消费成功的总数

### 生产者示例代码
```java
@Service
public class UserService {
    
    @Autowired
    private AsyncMessageProducer asyncMessageProducer;
    
    public void add(User user) {
        // balabalabala... 执行添加用户的业务逻辑
        
        asyncMessageProducer.send(user); //发送 User 消息至 Kafka 中，该方法不会抛出任何异常
    }
}
```

## 更多信息

* JDK 版本：1.8+
* 需要依赖的类库：
  * naivemonitor 1.0+
  * slf4j-log4j12 1.7.5+
  * kafka-clients 0.10.0.0+
* 更多 Kafka 信息：[Kafka 官方文档](http://kafka.apache.org/documentation/)
* 更多 NaiveMonitor 信息： [NaiveMonitor 项目 GitHub 地址](https://github.com/heimuheimu/naivemonitor)
