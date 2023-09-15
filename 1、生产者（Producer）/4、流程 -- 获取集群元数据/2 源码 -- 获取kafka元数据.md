

# 初始化 this.metadata

```java
KafkaProducer(ProducerConfig config,  
              Serializer<K> keySerializer,  
              Serializer<V> valueSerializer,  
              ProducerMetadata metadata,  
              KafkaClient kafkaClient,  
              ProducerInterceptors<K, V> interceptors,  
              Time time) {

	// 获取需要的第1个参数
	long retryBackoffMs = config.getLong(ProducerConfig.RETRY_BACKOFF_MS_CONFIG);   // "retry.backoff.ms" 

	// 获取需要的第2个参数
	String transactionalId = config.getString(ProducerConfig.TRANSACTIONAL_ID_CONFIG); // "transactional.id"
    this.clientId = config.getString(ProducerConfig.CLIENT_ID_CONFIG);                 // "client.id"
	LogContext logContext;  
	if (transactionalId == null)  
	    logContext = new LogContext(
		    String.format("[Producer clientId=%s] ", 
		    clientId)
		);  
	else  
	    logContext = new LogContext(
		    String.format("[Producer clientId=%s, transactionalId=%s] ", 
		    clientId, 
		    transactionalId)
		);  
	log = logContext.logger(KafkaProducer.class);  


	// 获取需要的第3个参数
	List<MetricsReporter> reporters = CommonClientConfigs.metricsReporters(clientId, config);
	... // 初始化 keySerializer
	... // 初始化 valueSerializer
	... // 初始化 interceptorList(拦截器)
	ClusterResourceListeners clusterResourceListeners = configureClusterResourceListeners(
		keySerializer,  
        valueSerializer, 
        interceptorList, 
        reporters
    );

	// 
	if (metadata != null) {  
	    this.metadata = metadata;  
	} else {  
	    this.metadata = new ProducerMetadata(
		    retryBackoffMs,  
	        config.getLong(ProducerConfig.METADATA_MAX_AGE_CONFIG),  
	        config.getLong(ProducerConfig.METADATA_MAX_IDLE_CONFIG),  
	        logContext,  
	        clusterResourceListeners,  
	        Time.SYSTEM);  
	    this.metadata.bootstrap(addresses);  
	}
}
```




```java
private ClusterAndWaitTime waitOnMetadata(String topic, Integer partition, long nowMs, long maxWaitMs) throws InterruptedException {  
    // add topic to metadata topic list if it is not there already and reset expiry  
    Cluster cluster = metadata.fetch();  
  
    if (cluster.invalidTopics().contains(topic))  
        throw new InvalidTopicException(topic);  
  
    metadata.add(topic, nowMs);  
  
    Integer partitionsCount = cluster.partitionCountForTopic(topic);  
    // Return cached metadata if we have it, and if the record's partition is either undefined  
    // or within the known partition range    if (partitionsCount != null && (partition == null || partition < partitionsCount))  
        return new ClusterAndWaitTime(cluster, 0);  
  
    long remainingWaitMs = maxWaitMs;  
    long elapsed = 0;  
    // Issue metadata requests until we have metadata for the topic and the requested partition,  
    // or until maxWaitTimeMs is exceeded. This is necessary in case the metadata    // is stale and the number of partitions for this topic has increased in the meantime.    long nowNanos = time.nanoseconds();  
    do {  
        if (partition != null) {  
            log.trace("Requesting metadata update for partition {} of topic {}.", partition, topic);  
        } else {  
            log.trace("Requesting metadata update for topic {}.", topic);  
        }  
        metadata.add(topic, nowMs + elapsed);  
        int version = metadata.requestUpdateForTopic(topic);  
        sender.wakeup();  
        try {  
            metadata.awaitUpdate(version, remainingWaitMs);  
        } catch (TimeoutException ex) {  
            // Rethrow with original maxWaitMs to prevent logging exception with remainingWaitMs  
            throw new TimeoutException(  
                    String.format("Topic %s not present in metadata after %d ms.",  
                            topic, maxWaitMs));  
        }  
        cluster = metadata.fetch();  
        elapsed = time.milliseconds() - nowMs;  
        if (elapsed >= maxWaitMs) {  
            throw new TimeoutException(partitionsCount == null ?  
                    String.format("Topic %s not present in metadata after %d ms.",  
                            topic, maxWaitMs) :  
                    String.format("Partition %d of topic %s with partition count %d is not present in metadata after %d ms.",  
                            partition, topic, partitionsCount, maxWaitMs));  
        }  
        metadata.maybeThrowExceptionForTopic(topic);  
        remainingWaitMs = maxWaitMs - elapsed;  
        partitionsCount = cluster.partitionCountForTopic(topic);  
    } while (partitionsCount == null || (partition != null && partition >= partitionsCount));  
  
    producerMetrics.recordMetadataWait(time.nanoseconds() - nowNanos);  
  
    return new ClusterAndWaitTime(cluster, elapsed);  
}
```
