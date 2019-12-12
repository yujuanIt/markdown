# Kafka 自定义指定消息partition策略规则及DefaultPartitioner源码分析

## 一.概述

kafka默认使用DefaultPartitioner类作为默认的partition策略规则,具体默认设置是在ProducerConfig类中(如下图)

![QkwrwR.png](https://s2.ax1x.com/2019/11/29/QkwrwR.png)

## 二.DefaultPartitioner.class 源码分析

### 1.类关系图

![Qk0ZnJ.png](https://s2.ax1x.com/2019/11/29/Qk0ZnJ.png)

### 2.源码分析

```java
public class DefaultPartitioner implements Partitioner {
	//缓存map key->topic  value->RandomNumber 随机数 
    private final ConcurrentMap<String, AtomicInteger> topicCounterMap = new ConcurrentHashMap<>();
	
	//实现Configurable接口里configure方法,
    public void configure(Map<String, ?> configs) {}

	//策略核心方法
    public int partition(String topic, Object key, byte[] keyBytes, Object value, byte[] valueBytes, Cluster cluster) {
    	//根据topic获取对应的partition
        List<PartitionInfo> partitions = cluster.partitionsForTopic(topic);
        int numPartitions = partitions.size();
        //如果key是null的话
        if (keyBytes == null) {
            int nextValue = nextValue(topic);
            //获取可用的分区数量
            List<PartitionInfo> availablePartitions = cluster.availablePartitionsForTopic(topic);
            //如果存在可用的分区
            if (availablePartitions.size() > 0) {
            	//消息随机分布到topic的可用partition中
                int part = Utils.toPositive(nextValue) % availablePartitions.size();
                return availablePartitions.get(part).partition();
            } else {
				//不存在可用分区 随机分配一个不可用的partition中            		
                return Utils.toPositive(nextValue) % numPartitions;
            }
        } else {
        	//使用自己的 hash 算法对 key 取 hash 值，使用 hash 值与 partition 数量取模，从而确定发送到哪个分区。
            return Utils.toPositive(Utils.murmur2(keyBytes)) % numPartitions;
        }
    }

    private int nextValue(String topic) {
        //获取topic 获取对应的随机数
        AtomicInteger counter = topicCounterMap.get(topic);
        if (null == counter) {
            //获取一个随机值
            counter = new AtomicInteger(ThreadLocalRandom.current().nextInt());
            //缓存到topicCounterMap中
            AtomicInteger currentCounter = topicCounterMap.putIfAbsent(topic, counter);
            if (currentCounter != null) {
                counter = currentCounter;
            }
        }
        //获取 并且实现自增,最终效果是实现轮训插入partition
        return counter.getAndIncrement();
    }

    public void close() {}

}
```





## 三.自定义Partition

自定义selfPartitioner类,并且实现Partitioner接口,重写partition和close方法

```java
public class SelfPartitioner implements Partitioner {

    @Override
    public int partition(String topic, Object key, byte[] keyBytes, Object value, byte[] valueBytes, Cluster cluster) {
        //TODO 自定义数据分区策略
        return 0;
    }

    @Override
    public void close() {

    }

    @Override
    public void configure(Map<String, ?> configs) {

    }
}
```

## 四.生产者中使用自定义的Partition

在初始化producer时候,配置项中指定对应的partitioner

```
props.put("partitioner.class", "org.jake.partitioner.selfPartitioner");
```



