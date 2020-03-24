---
layout: post
title:  Implementing a Health Check Mechanism for Kafka
categories: scala
tags: [scala, kafka, health check]
---


Kafka is a well known distributed streaming platform that stores data in topics and provides scalability with topic partitions. 

At Veon, we use Kafka extensively to publish and consume events from across different microservices. As almost all our microservices emit/consume events to/from Kafka, we wanted to build a Healthcheck mechanism for checking connectivity to Kafka.

Some key concepts about Kafka used in the health-check  
*Partition:* Kafka provides parallelism by storing its data across multiple partitions.   
*Consumer Group:* A subscriber to a particular topic.  
*Consumers in a Consumer Group:* Partitions of a topic are balanced between the consumers of a consumer group. This enables parallel consumption of data from the topic. The number of consumers can scale horizontally upto the number of partitions.  
*Offset:* Each message in a partition is indexed with an identifier called an offset. Kafka stores the last committed offset for each partition for every consumer group. This offset determines where to start processing in case an instance(consumer) fails and restarts(or rebalanced).  

Some important configuration parameters for the health-check  
<span style="color:red">`offset.retention.minutes` </span>:  The time for which kafka remembers the offset for a partition. Defaults to 24 hours.   
<span style="color:red">`auto.offset.reset` </span>: In case a new consumer-group comes in or *offset.retention.minutes* has elapsed, there needs to be a way to start processing messages. Two options are provided either start processing from the start(*earliest*) or at the end(*latest*). Defaults to *earliest* to not miss processing data.  


#### <span style="color:blue">Building a health-check mechanism</span>  
At Veon, all application instances run on docker containers in swarm mode.   
Docker health-check can be thought of as just a simple curl call to the `/health` endpoint inside the container. 
    
    HEALTHCHECK CMD curl --fail http://localhost/health     

On this `/health` a variety of HealthChecks can be plugged as required. One of them will be the KafkaHealthCheck.  

##### <span style="color:blue">Kafka health-check</span>     
There are multiple ways to design a health check for Kafka. The simplest that we decided to implement was to produce a message to Kafka and consume it within a threshold time.  

**1. A Singleton KafkaHealthCheck Object**  

{% highlight scala %}
object KafkaHealthCheck {
  def apply(checkName: String = "kafka-health-check")(implicit system: ActorSystem): StatusCheck[Future] = {
   	start(checkName, HealthCheckConfig.apply)
  }
}
{% endhighlight %}

**2. Configuring our healthchecks**  

{% highlight scala %}
case class HealthCheckConfig(
  tickTime: FiniteDuration,
  failureFactor: Double,
  serviceName: String,
  identifier: String
)
{% endhighlight %}

What do each these healthcheck Config mean?     

*tickTime* - How often to produce a message to check the healthiness  
*failureFactor*- How many ticks should we wait before the alarm triggers.  
*serviceName* - For logging purposes.  
*Identifier* - Instance identifier which is also used for the Consumer-Group and needs to be unique among the active offsets remembered.  

**3. Health Check Message(Ping) Producer**  

{% highlight scala %}
def runHealthCheckProducer(
    producerSettings: ProducerSettings[String, String],
    healthCheckConfig: HealthCheckConfig
  )(implicit clock: Clock, materializer: Materializer): Unit = {
    Source
      .tick(
        healthCheckConfig.tickTime,
        healthCheckConfig.tickTime,
        ()
      )
      .map(_ => HealthCheckPing(healthCheckConfig.serviceName, healthCheckConfig.identifier, clock.instant()))
      .map(HealthCheckPing.toJson)
      .via(debugLog(msg => s"Sending health check $msg to kafka"))
      .map(new ProducerRecord[String, String](topic, _))
      .to(Producer.plainSink(producerSettings))
      .run()
  }
{% endhighlight %}

This produces a *HealthCheckPing* every tick seconds into the health-check topic. The Ping itself encapsulates *Who and When* of this message.

**4. Health Check Ping Consumer**  

{% highlight scala %}
 private def runHealthCheckConsumer(
    consumerSettings: ConsumerSettings[String, String],
    healthCheckConfig: HealthCheckConfig
  )(onCheckReceive: HealthCheckPing => Unit)(implicit system: ActorSystem, materializer: Materializer): Unit = {
    committableSource(consumerSettings, Subscriptions.topics(topic))
      .alsoTo(kafkaCommitSink)
      .map(msg => HealthCheckPing.fromJson(msg.record.value()))
      .collect {
        case Some(msg: HealthCheckPing) if msg.isMine(healthCheckConfig) => msg
      }
      .via(debugLog(msg => s"Received health check from kafka $msg"))
      .toMat(Sink.foreach(onCheckReceive))(Keep.left)
      .run()
  }
{% endhighlight %}

The consumer subscribes to the health-check topic and filters only messages that it produced (based on the identifier encapsulated in the Ping). 
There is a callback(*onCheckReceive*) invoked when a successful Ping was received.

**5. The HealthCheck Ping**  

{% highlight scala %}
private[healthcheck] case class HealthCheckPing(serviceName: String, identifier: String, timestamp: Instant) {
  def isMine(config: HealthCheckConfig): Boolean = serviceName == config.serviceName && identifier == config.identifier
}
{% endhighlight %}

**6. Connecting the pieces together**  

We use a simple actor with state to hold the time in which a last successful produced message was received. 
{% highlight scala %}
def start(
    checkName: String,
    healthCheckConfig: HealthCheckConfig
  )(implicit system: ActorSystem, clock: Clock = Clock.systemUTC()): StatusCheck[Future] = {
    implicit val mat: ActorMaterializer = materializer
    import system.dispatcher

    val producerConsumerSettings = ProducerConsumerSettings(healthCheckConfig)

    var lastHealthCheckAt = clock.instant()

    info(s"Starting kafka health check with config $healthCheckConfig")
    runHealthCheckProducer(producerConsumerSettings.producerSettings, healthCheckConfig)
    runHealthCheckConsumer(producerConsumerSettings.consumerSettings, healthCheckConfig)(
      message => lastHealthCheckAt = message.timestamp
    )
  }
{% endhighlight %}

The callback on a successful ping reception updates the *lastHealthCheckAt* state.

**7. The Health-check check**  
{% highlight scala %}
check[Future](checkName) {
    Future.successful {
        val failAfter = healthCheckConfig.tickTime * healthCheckConfig.failureFactor
        if (lastHealthCheckAt.isAfter(clock.instant().minus(failAfter.toSeconds, ChronoUnit.SECONDS))) Ok
        else Ko(s"Last health check received at $lastHealthCheckAt and it's older than $failAfter")
    }
}
{% endhighlight %}

The *check* call is invoked when the `/health` endpoint is invoked by Docker. *`healthCheckConfig.tickTime * healthCheckConfig.failureFactor`* determines the threshold to wait. 
If the last successful healthcheck was received before this threshold, then it returns failure(KO) else everything is OK.

Some points to remember while setting up the healthcheck. 
It is necessary to set the *auto.offset.reset=latest* for this topic. Since every new instance is its own consumer-group, kafka resorts to the default *earliest* strategy and reads all messages from start. This can cause the threshold to elaspse, triggering Docker swarm to replace unhealthy containers. This then repeats the process of reading the message from start by the new containers and elapsing the threshold. And the cycle of failures & restarts goes on indefinitely.

As our services were running over docker-swarm, we decided to use the container-id provided by docker as identifer. 
[Note: We saw that *offset.retention.minutes* defaults to 24 hours]. Till this 24 hours, the consumer-group id should not repeat else, it might start receiving older(some other) messages and might appear unhealthy causing the indefinite restarts.

That's all Folks! This was a simple health check on Kafka added at Veon.