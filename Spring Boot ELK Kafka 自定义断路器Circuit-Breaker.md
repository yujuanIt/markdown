## Spring Boot ELK Kafka 自定义断路器Circuit-Breaker 

## 一.需求说明

​	微服务框架需要日志收集,包括日志的清洗分析过滤等,常见的日志系统是ELK.业务系统通过ELK组件,将日志通过logback的方式写入kafka,logstash对kafka的日志进行清洗过滤,最后统一进入kinbana进行日志的分析和汇总.						  

​	kafka作为中间件,正常是不可以影响应用状态的,但是在应用启动或者运行过程中,通常会发生kafka无法接受消息,或者挂掉了的情况,导致每次写入日志时都会初始化producer并且初始化失败,造成很多异常日志信息(严重的日志会把整个磁盘写满),就会影响应用的状态.例如下面的这个图(kafka在初始化时连接配置写错了,初始化不成功,导致一直在初始化bean)

![Qp4QFs.png](https://s2.ax1x.com/2019/11/27/Qp4QFs.png)

​	所以正确的做法就是加入熔断(Circuit-Breaker): 如果kafka挂掉以后,可以通过熔断的方式执行熔断策略,降低对业务系统的影响

## 二.实现

### 1.自定义KafkaAppender

​	在核心方法append方法中,先判断熔断器的状态,如果是开启状态,则直接就对应的失败策略failedDeliveryCallback.onFailedDelivery(e, null);如果未开启,则初始化producer,初始化失败则调用actFailed方法记录失败次数+1,并且走对应失败策略;初始化成功,调用	actSuccess方法 记录成功次数+1,并且发送消息到kafka中

```
import ch.qos.logback.classic.spi.ILoggingEvent;
import ch.qos.logback.core.Appender;
import ch.qos.logback.core.spi.AppenderAttachableImpl;
import com.github.danielwegener.logback.kafka.KafkaAppenderConfig;
import com.github.danielwegener.logback.kafka.delivery.FailedDeliveryCallback;
import org.apache.kafka.clients.producer.KafkaProducer;
import org.apache.kafka.clients.producer.Producer;
import org.apache.kafka.clients.producer.ProducerConfig;
import org.apache.kafka.clients.producer.ProducerRecord;
import org.apache.kafka.common.KafkaException;
import org.apache.kafka.common.serialization.ByteArraySerializer;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

import java.util.HashMap;
import java.util.Iterator;
import java.util.concurrent.ConcurrentLinkedQueue;

/**
 * 自定义Appender
 * @param <E>
 */
public class KafkaAppender<E> extends KafkaAppenderConfig<E> {
    private static final String KAFKA_LOGGER_PREFIX = "org.apache.kafka.clients";
    public static final Logger logger = LoggerFactory.getLogger(KafkaAppender.class);
    private LazyProducer lazyProducer = null;
    private final AppenderAttachableImpl<E> aai = new AppenderAttachableImpl<E>();
    private final ConcurrentLinkedQueue<E> queue = new ConcurrentLinkedQueue<E>();

    //加载断路器 
    CircuitBreaker circuitBreaker = CircuitBreakerFactory.buildCircuitBreaker("KafkaAppender-c", 3, 2, 20000);

    private final FailedDeliveryCallback<E> failedDeliveryCallback = new FailedDeliveryCallback<E>() {
        @Override
        public void onFailedDelivery(E evt, Throwable throwable) {
            aai.appendLoopOnAppenders(evt);
        }
    };

    public KafkaAppender() {
        // setting these as config values sidesteps an unnecessary warning (minor bug in KafkaProducer)
        addProducerConfigValue(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG, ByteArraySerializer.class.getName());
        addProducerConfigValue(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG, ByteArraySerializer.class.getName());
    }

    @Override
    public void doAppend(E e) {
        ensureDeferredAppends();
        if (e instanceof ILoggingEvent && ((ILoggingEvent) e).getLoggerName().startsWith(KAFKA_LOGGER_PREFIX)) {
            deferAppend(e);
        } else {
            super.doAppend(e);
        }
    }

    @Override
    public void start() {
        //检查配置参数是否完整
        if (!checkPrerequisites()) return;
        //初始化producer bean
        lazyProducer = new LazyProducer();
        super.start();
    }

    @Override
    public void stop() {
        super.stop();
        if (lazyProducer != null && lazyProducer.isInitialized()) {
            try {
                lazyProducer.get().close();
            } catch (KafkaException e) {
                this.addWarn("Failed to shut down kafka producer: " + e.getMessage(), e);
            }
            lazyProducer = null;
        }
    }

    @Override
    public void addAppender(Appender<E> newAppender) {
        aai.addAppender(newAppender);
    }

    @Override
    public Iterator<Appender<E>> iteratorForAppenders() {
        return aai.iteratorForAppenders();
    }

    @Override
    public Appender<E> getAppender(String name) {
        return aai.getAppender(name);
    }

    @Override
    public boolean isAttached(Appender<E> appender) {
        return aai.isAttached(appender);
    }

    @Override
    public void detachAndStopAllAppenders() {
        aai.detachAndStopAllAppenders();
    }

    @Override
    public boolean detachAppender(Appender<E> appender) {
        return aai.detachAppender(appender);
    }

    @Override
    public boolean detachAppender(String name) {
        return aai.detachAppender(name);
    }

    @Override
    protected void append(E e) {
        // encode 逻辑
        final byte[] payload = encoder.encode(e);
        final byte[] key = keyingStrategy.createKey(e);
        final ProducerRecord<byte[], byte[]> record = new ProducerRecord<byte[], byte[]>(topic, key, payload);

        // 如果还是熔断状态 则不初始化producer 直接走失败处理
        if (circuitBreaker.isOpen()) {
            failedDeliveryCallback.onFailedDelivery(e, null);
            return;
        }
        //初始化producer
        Producer producer = lazyProducer.get();
        if (producer == null) {
            //初始化失败 错误+1 并且走熔断
            circuitBreaker.actFailed();
            logger.error("kafka producer is null");
            failedDeliveryCallback.onFailedDelivery(e, null);
            return;
        }
        //成功一次 success+1
        circuitBreaker.actSuccess();
        // 核心发送方法
        deliveryStrategy.send(lazyProducer.get(), record, e, failedDeliveryCallback);
    }

    protected Producer<byte[], byte[]> createProducer() {
        return new KafkaProducer<>(new HashMap<>(producerConfig));
    }

    private void deferAppend(E event) {
        queue.add(event);
    }
    // drains queue events to super

    private void ensureDeferredAppends() {
        E event;
        while ((event = queue.poll()) != null) {
            super.doAppend(event);
        }
    }

    private class LazyProducer {
        private volatile Producer<byte[], byte[]> producer;
        private boolean initialized;

        public Producer<byte[], byte[]> get() {
            Producer<byte[], byte[]> result = this.producer;
            if (result == null) {
                synchronized (this) {
                    if (!initialized) {
                        result = this.producer;
                        if (result == null) { // 注意 这里initialize可能失败，比如传入servers为非法字符，返回producer为空，所以只用initialized标记来确保不进行重复初始化，而避免不断出错的初始化
                            this.producer = result = this.initialize();
                            initialized = true;
                        }
                    }
                }
            }
            return result;
        }

        protected Producer<byte[], byte[]> initialize() {
            Producer<byte[], byte[]> producer = null;
            try {
                producer = createProducer();
            } catch (Exception e) {
                addError("error creating producer", e);
            }
            return producer;
        }

        public boolean isInitialized() {
            return producer != null;
        }
    }
}

```

### 2.自定义Delivery策略

​	在这里也需要加入熔断策略,发送成功调用actSuccess方法记录成功次数+1,发送失败调动actFailed方法记录失败次数+1,并且执行对应的失败粗略.

```
import com.github.danielwegener.logback.kafka.delivery.DeliveryStrategy;
import com.github.danielwegener.logback.kafka.delivery.FailedDeliveryCallback;
import org.apache.kafka.clients.producer.Producer;
import org.apache.kafka.clients.producer.ProducerRecord;
import org.apache.kafka.common.KafkaException;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

public class SelfDeliveryStrategy implements DeliveryStrategy {
    private final static Logger logger = LoggerFactory.getLogger(SelfDeliveryStrategy.class);
    CircuitBreaker circuitBreaker = CircuitBreakerFactory.buildCircuitBreaker("KafkaAppender-c", 3, 2, 20000);

    public <K, V, E> boolean send(Producer<K, V> producer, ProducerRecord<K, V> record, final E event, final FailedDeliveryCallback<E> failedDeliveryCallback) {
        if (circuitBreaker.isNotOpen()) {
            try {
                producer.send(record, (metadata, exception) -> {
                    if (exception != null) {
                        circuitBreaker.actFailed();
                        failedDeliveryCallback.onFailedDelivery(event, exception);
                        logger.error("kafka producer send log error", exception);
                    } else {
                        circuitBreaker.actSuccess();
                    }
                });
                return true;
            } catch (KafkaException e) {
                circuitBreaker.actFailed();
                failedDeliveryCallback.onFailedDelivery(event, e);
                logger.error("kafka send log error", e);
                return false;
            }
        } else {
            logger.error("kafka log circuitBreaker open");
            return false;
        }
    }

}
```

### 3.自定义断路器

​	断路器的bean,记录断路器的各种状态信息,包括名称,状态,失败阀值,时间,成功次数,失败次数等

```
import com.alibaba.fastjson.JSONObject;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

import java.util.HashMap;
import java.util.Map;
import java.util.concurrent.atomic.AtomicInteger;

public class CircuitBreaker {
    private static final Logger logger = LoggerFactory.getLogger(CircuitBreaker.class);
    /**
     * 熔断器名称
     */
    private String name;
    /**
     * 熔断器状态
     */
    private CircuitBreakerState state;
    /**
     * 失败次数阀值
     */
    private int failureThreshold;
    /**
     * 熔断状态时间窗口
     */
    private long timeout;
    /**
     * 失败次数
     */
    private AtomicInteger failureCount;
    /**
     * 成功次数 （并发不准确）
     */
    private int successCount;
    /**
     * 半开时间窗口里连续成功的次数
     */
    private AtomicInteger consecutiveSuccessCount;
    /**
     * 半开时间窗口里连续成功的次数阀值
     */
    private int consecutiveSuccessThreshold;

    public CircuitBreaker(String name, int failureThreshold, int consecutiveSuccessThreshold, long timeout) {
        if (failureThreshold <= 0) {
            failureThreshold = 1;
        }
        if (consecutiveSuccessThreshold <= 0) {
            consecutiveSuccessThreshold = 1;
        }
        if (timeout <= 0) {
            timeout = 10000;
        }
        this.name = name;
        this.failureThreshold = failureThreshold;
        this.consecutiveSuccessThreshold = consecutiveSuccessThreshold;
        this.timeout = timeout;
        this.failureCount = new AtomicInteger(0);
        this.consecutiveSuccessCount = new AtomicInteger(0);
        state = new CloseCircuitBreakerState(this);
    }

    public void increaseFailureCount() {
        failureCount.addAndGet(1);
    }

    public void increaseSuccessCount() {
        successCount++;
    }

    public void increaseConsecutiveSuccessCount() {
        consecutiveSuccessCount.addAndGet(1);
    }

    public boolean increaseFailureCountAndThresholdReached() {
        return failureCount.addAndGet(1) >= failureThreshold;
    }

    public boolean increaseConsecutiveSuccessCountAndThresholdReached() {
        return consecutiveSuccessCount.addAndGet(1) >= consecutiveSuccessThreshold;
    }

    public boolean isNotOpen() {
        return !isOpen();
    }

    /**
     * 熔断开启 关闭保护方法的调用 * @return
     */
    public boolean isOpen() {
        return state instanceof OpenCircuitBreakerState;
    }

    /**
     * 熔断关闭 保护方法正常执行
     *
     * @return
     */
    public boolean isClose() {
        return state instanceof CloseCircuitBreakerState;
    }

    /**
     * 熔断半开 保护方法允许测试调用
     *
     * @return
     */
    public boolean isHalfClose() {
        return state instanceof HalfOpenCircuitBreakerState;
    }

    public void transformToCloseState() {
        state = new CloseCircuitBreakerState(this);
    }

    public void transformToHalfOpenState() {
        state = new HalfOpenCircuitBreakerState(this);
    }

    public void transformToOpenState() {
        state = new OpenCircuitBreakerState(this);
    }

    /**
     * 重置失败次数
     */
    public void resetFailureCount() {
        failureCount.set(0);
    }

    /**
     * 重置连续成功次数
     */
    public void resetConsecutiveSuccessCount() {
        consecutiveSuccessCount.set(0);
    }

    public long getTimeout() {
        return timeout;
    }

    /**
     * 判断是否到达失败阀值 * @return
     */
    protected boolean failureThresholdReached() {
        return failureCount.get() >= failureThreshold;
    }

    /**
     * 判断连续成功次数是否达到阀值 * @return
     */
    protected boolean consecutiveSuccessThresholdReached() {
        return consecutiveSuccessCount.get() >= consecutiveSuccessThreshold;
    }

    /**
     * 保护方法失败后操作
     */
    public void actFailed() {
        state.actFailed();
    }

    /**
     * 保护方法成功后操作
     */
    public void actSuccess() {
        state.actSuccess();
    }
}
```

### 4.自定义断路器状态相关类

####  通用接口

````

public interface CircuitBreakerState {
    /**
     * 保护方法失败后操作
     */
    void actFailed();

    /**
     * 保护方法成功后操作
     */
    void actSuccess();
}
````

#### 抽象断路器状态类

````

public abstract class AbstractCircuitBreakerState implements CircuitBreakerState {
    protected CircuitBreaker circuitBreaker;

    public AbstractCircuitBreakerState(CircuitBreaker circuitBreaker) {
        this.circuitBreaker = circuitBreaker;
    }

    @Override
    public void actFailed() {
        circuitBreaker.increaseFailureCount();
    }

    @Override
    public void actSuccess() {
        circuitBreaker.increaseSuccessCount();
    }
}

````

#### 关闭状态实现类

​	重写actFailed方法,次数+1,并且判断失败次数是否到达阈值,到达后开启断路器

```
public class CloseCircuitBreakerState extends AbstractCircuitBreakerState {
    public CloseCircuitBreakerState(CircuitBreaker circuitBreaker) {
        super(circuitBreaker);
        circuitBreaker.resetFailureCount();
        circuitBreaker.resetConsecutiveSuccessCount();
    }

    @Override
    public void actFailed() {
        // 进入开启状态
        if (circuitBreaker.increaseFailureCountAndThresholdReached()) {
            circuitBreaker.transformToOpenState();
        }
    }
}
```

#### 半开状态实现类

​	重新actFailed方法和actSuccess方法.

​	actFailed:如果在半开状态时发生失败,则立即打开断路器

​	actSuccess:次数+1,并且判断是否到达关闭熔断的阈值,如果达到成功的次数,就关闭熔断

```

public class HalfOpenCircuitBreakerState extends AbstractCircuitBreakerState {
    public HalfOpenCircuitBreakerState(CircuitBreaker circuitBreaker) {
        super(circuitBreaker);
        circuitBreaker.resetConsecutiveSuccessCount();
    }

    @Override
    public void actFailed() {
        super.actFailed();
        circuitBreaker.transformToOpenState();
    }

    @Override
    public void actSuccess() {
        // 达到成功次数的阀值 关闭熔断
        if (circuitBreaker.increaseConsecutiveSuccessCountAndThresholdReached()) {
            circuitBreaker.transformToCloseState();
        }
    }
}
```

#### 断路器开启状态实现类

​	断路器开启以后,根据配置的超时时间,执行一个Timer定时任务,到期自动关闭断路器

````
public class OpenCircuitBreakerState extends AbstractCircuitBreakerState {
    public OpenCircuitBreakerState(CircuitBreaker circuitBreaker) {
        super(circuitBreaker);
        final Timer timer = new Timer();
        timer.schedule(new TimerTask() {
            @Override
            public void run() {
                circuitBreaker.transformToHalfOpenState();
                timer.cancel();
            }
        }, circuitBreaker.getTimeout());
    }

````

#### 断路器工厂类

​	使用map来维护熔断器,通过工厂模式直接获取

```
public class CircuitBreakerFactory {

    /**
     * 用来存储熔断器的map集合 通过工程模式直接获取
     */
    private static ConcurrentHashMap<String, CircuitBreaker> circuitBreakerMap = new ConcurrentHashMap();

    public CircuitBreaker getCircuitBreaker(String name) {
        CircuitBreaker circuitBreaker = circuitBreakerMap.get(name);
        return circuitBreaker;
    }

    /**
     * @param name                        唯一名称
     * @param failureThreshold            失败次数阀值
     * @param consecutiveSuccessThreshold 时间窗内成功次数阀值
     * @param timeout                     时间窗
     *                                    1.close状态时 失败次数>=failureThreshold，进入open状态
     *                                    2.open状态时每隔timeout时间会进入halfOpen状态
     *                                    3.halfOpen状态里需要连续成功次数达到consecutiveSuccessThreshold，
     *                                    4.即可进入close状态，出现失败则继续进入open状态
     * @return
     */
    public static CircuitBreaker buildCircuitBreaker(String name, int failureThreshold, int consecutiveSuccessThreshold, long timeout) {
        CircuitBreaker circuitBreaker = new CircuitBreaker(name, failureThreshold, consecutiveSuccessThreshold, timeout);
        circuitBreakerMap.put(name, circuitBreaker);
        return circuitBreaker;
    }
}
```

### 5.logback.xml配置 

#### 	不在使用默认的KafkaAppender和DeliveryStrategy,使用自定义的KafkaAppender和SelfDeliveryStrategy

```

<?xml version="1.0" encoding="UTF-8"?>
<configuration>
     <property name="CONSOLE_LOG_PATTERN"
                  value="%date{yyyy-MM-dd HH:mm:ss} | %highlight(%-5level) | %boldYellow(%thread) | %boldGreen(%logger) | %msg%n"/>
  
    <springProperty scope="context" name="logger.kafka.server" source="logger.kafka.server"/>
    <appender name="stdout" class="ch.qos.logback.core.ConsoleAppender">
        <encoder>
            <pattern>${CONSOLE_LOG_PATTERN}</pattern>
        </encoder>
    </appender>
    <appender name="file"
              class="ch.qos.logback.core.rolling.RollingFileAppender">
        <file>../logs/${project.artifactId}-service.log</file>
        <rollingPolicy class="ch.qos.logback.core.rolling.TimeBasedRollingPolicy">
            <FileNamePattern>../logs/${project.artifactId}-service-%d{yyyy-MM-dd}.log</FileNamePattern>
            <maxHistory>60</maxHistory>
        </rollingPolicy>
        <encoder>
            <pattern>${CONSOLE_LOG_PATTERN}</pattern>
        </encoder>
    </appender>

	<!-- 失败策略 发送到对应的文件 -->
    <appender name="service-log-kafka-lose" class="ch.qos.logback.core.rolling.RollingFileAppender">
        <file>./logs/kafka/lose_log.log</file>
        <rollingPolicy class="ch.qos.logback.core.rolling.TimeBasedRollingPolicy">
            <FileNamePattern>./logs/kafka/service_lose_log%d{yyyyMMdd}.log.zip</FileNamePattern>
            <maxHistory>30</maxHistory>
        </rollingPolicy>
        <encoder>
            <pattern>
                <pattern>
                    {
                    "Level": "%level",
                    "ServiceName": "Test1",
                    "Pid": "${PID:-}",
                    "Thread": "%thread",
                    "Class": "%logger{40}",
                    "Rest": "%message"
                    }
                </pattern>
            </pattern>
        </encoder>
    </appender>
    <!-- kafka的appender配置 -->
    <appender name="kafka" class="org.jake.kafkaAppender.KafkaAppender">
        <encoder class="net.logstash.logback.encoder.LoggingEventCompositeJsonEncoder">
            <providers>
                <timestamp>
                    <timeZone>UTC</timeZone>
                </timestamp>
                <pattern>
                    <pattern>
                        {
                        "Level": "%level",
                        "ServiceName": "Test1",
                        "Pid": "${PID:-}",
                        "Thread": "%thread",
                        "Class": "%logger{40}",
                        "Rest": "%message"
                        }
                    </pattern>
                </pattern>
            </providers>
        </encoder>
        <topic>service_logs</topic>
        <keyingStrategy class="com.github.danielwegener.logback.kafka.keying.ThreadNameKeyingStrategy"/>
        <deliveryStrategy class="org.jake.delivery.SelfDeliveryStrategy"/>

        <!-- Optional parameter to use a fixed partition -->
        <!-- <partition>0</partition> -->

        <!-- Optional parameter to include log timestamps into the kafka message -->
        <!-- <appendTimestamp>true</appendTimestamp> -->

        <!-- each <producerConfig> translates to regular kafka-client config (format: key=value) -->
        <!-- producer configs are documented here: https://kafka.apache.org/documentation.html#newproducerconfigs -->
        <!-- bootstrap.servers is the only mandatory producerConfig -->
        <producerConfig>bootstrap.servers=${logger.kafka.server}</producerConfig>
        <!-- don't wait for a broker to ack the reception of a batch.  -->
        <producerConfig>acks=0</producerConfig>
        <!-- wait up to 1000ms and collect log messages before sending them as a batch -->
        <producerConfig>linger.ms=1000</producerConfig>
        <producerConfig>reconnect.backoff.ms=1000</producerConfig>
        <producerConfig>block.on.buffer.full=false</producerConfig>
        <!-- even if the producer buffer runs full, do not block the application but start to drop messages -->
        <!--<producerConfig>max.block.ms=0</producerConfig>-->

        <!-- 如果kafka不可用则输出到控制台 -->
        <appender-ref ref="service-log-kafka-lose"/>
    </appender>

	<root level="INFO">
            <appender-ref ref="stdout"/>
            <appender-ref ref="file"/>
            <appender-ref ref="kafka"/>
	</root>
</configuration>
```
### 附
#### pom相关引用

```

 		<dependency>
            <groupId>com.github.danielwegener</groupId>
            <artifactId>logback-kafka-appender</artifactId>
            <version>0.2.0-RC2</verison>
        </dependency>
        <dependency>
            <groupId>net.logstash.logback</groupId>
            <artifactId>logstash-logback-encoder</artifactId>
            <version>5.2</verison>
        </dependency>
        <dependency>
            <groupId>org.apache.kafka</groupId>
            <artifactId>kafka-clients</artifactId>
            <version>2.2.1</verison>
        </dependency>
        
```

### 

> 参考Circuit Breaker 设计模式（断路模式） <https://www.oschina.net/question/54100_2285459>