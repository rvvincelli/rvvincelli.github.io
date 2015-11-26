---
layout: post
title: Kafka & Spark
---

##Integrating Kafka with Spark - hands on

As you all know already [Apache Kafka](http://kafka.apache.org) is a distributed publisher-subscriber system capable of handling quite heavy loads and throughputs too - which turns out to be not
just another *BDBC* (Big Data Big Claim) given that the project started at [Linkedin](http://www.linkedin.com), is widely adopted and supported by [Cloudera](http://www.cloudera.com), among others. [Spark](http://spark.apache.org) is getting real momentum these days so no need
for introduction, and a really interesting module is [Spark Streaming](http://spark.apache.org/streaming), which allows to feed the application a live data stream to be processed in a batch-window fashion with the usual Spark functional
-like operations. The integration comes quite easy and we go through a small example now.

###Kafka what?

In a nutshell, Kafka is a producer-consumer system, the producer, identified by some ID, sends a message labeled with a topic to the broker, which gets in turn subscriptions from a consumer. You can
of course have multiple instances for each of these three roles. The basic unity of parallelism in Kafka is the partition number for topic (`num.partitions`, configurable on the cluster); briefly, more
partitions lead to more throughput as Kafka allows only one single thread to attach to a given partition. An important point is that the message set is totally ordered with respect to a partition
only. See the [documentation](http://kafka.apache.org/documentation.html) for more info.

Here's a picture:

 1. the producer starts sending out messages under a certain topic
 2. the cluster receives such messages and stores them in one of the partitions (the default partitions number is configurable, see above)
 3. a consumer connects to the cluster and asks for the messages for a topic; as the server feeds the messages it keeps track of a counter, the *offset*, associated with the topic and the consumer group identifier
 4. when a consumer connects, if an offset is available for its group, it is fed the messages from the known offset on; if not, `auto.offset.reset` in the consumer defines what to do (not exposed now)

But let's check out a more complex scenario too, taking into account the particular Kafka protocol a little too.

At Kafka protocol level, a consumer may be characterized by a topic name, whereas a consumer by a topic name together with a group id.
The simplest case is that of a producer `P` of messages of topic `T` running in parallel with a consumer `C` of topic `T` from the group `G`, both correctly connected to the the Kafka cluster, let them run
for a while. Now let's say the consumer is gracefully restarted. As the broker was sending out to `C` the messages produced by `P` it was keeping track of the id (called *offset* in Kafka) for `C`, or more
precisely for `G`, `T`'s group. This fact guarantees the following: when `C` is back and connects to the broker again it will be fed the messages from the last one read by any of its peers in the consumer
group `G` on. This is true at the high level, even if under the hood Kafka uses partitioning, but the takeaway is that the resurrected consumer will be fed *virgin* so to say, unconsumed, messages only.
Now a new consumer `D` pops up, still asking for the topic `T` but hailing from a different group, `H`, asking for message number 1. A topic with a number of messages already exists, or better one offset
exists on the topic already, somebody is actually consuming it, namely the guys from the group `G` above. The broker notifies the consumer about this, which reacts according to the behavior specified in
`auto.offset.reset`:
 
 * if `smallest` is set the consumer will request the minimum between the existing broker-side offset and its own, 1 in this case, so it will be fed the messages from offset 1 on
 * `greatest` works dually, so in this case the consumer will get new incoming messages only
 * anything else throws an exception  


Let's focus now on the producer side, which we will show with the native Kafka APIs. All of the code is available [here](https://github.com/rvvincelli/kafkasparkexample).

First what will our messages look like? Here is a little case class: case classes are immutable by default and are therefore a good pick for distributed messages - for example this is the rule in [Akka](http://akka.io/).

```scala
case class RichEvent(content: String)
```

A producer may post freely to topics, so this is not part of its configuration, but a number of other properties must be configured:

```scala
ProducerConfig.BOOTSTRAP_SERVERS_CONFIG      -> "kafkabroker:9092",
ProducerConfig.ACKS_CONFIG                   -> "3",
ProducerConfig.CLIENT_ID_CONFIG              -> "clientid",
ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG   -> "org.apache.kafka.common.serialization.StringSerializer",
ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG -> "io.github.rvvincelli.blogpost.kafkaspark.RichEventSerializer"
```

`BOOTSTRAP_SERVERS_CONFIG`: the Kafka broker the producer talks to after startup in order to coordinate the information needed to post its messages; it acts as configuration bootstrap node as the
as the name suggests; multiple values can be specified

`ACKS_CONFIG`: Kafka supports different styles of acknowledgement:

 * `request.required.acks` set to `0`; *no ack*, the producer sends messages in a *fire and forget* fashion, thus not caring about the actual delivery status; this is the default behavior 
 * `request.required.acks` is `1`; master replica only, the producer only waits for master replica confirmation feedback; the acknowledged message might still go lost anyway in case the master fails
    without propagating the message to the rest of the brokers; this is `n`-generalizable in the sense that we may choose to wait for the acknowledgement from `n` brokers, master included  
 * `request.required.acks` is `-1`; *all*, we want all of the nodes to confirm successfully, which implies that, once we get the ackowledgement, as long as at least one broker is up and running, the
    message is always available to the consumers  

It is important to notice that acknowledgement feeds may interfere with the order of the messages as received on the broker side. Also, the management is all internal but in determining the behavior
the following are important:

  * `retries` defines how many attempts the producer makes after a transient (non fatal) send error, eg an acknowledgement failure notice from the server; a backoff interval can be defined as well via
   `retry.backoff.ms`; enabling resend breaks the property that messages are always available in-order at the broker site for the producers to consume, as well as at-most-once delivery
  * `request.timeout.ms` defines the time the master broker waits for the fulfillment of the acknowledgement mode chosen by the client, returning an error to it once this timeout elapses

* `CLIENT_ID`: the client ID does not play a role in the Kafka protocol yet it may be useful for debug; it should identify the producer instance univocally

* `KEY_SERIALIZER_CLASS_CONFIG`, `VALUE_SERIALIZER_CLASS_CONFIG`: Kafka messages are key-value pairs where the key is optional, the producer needs to know how to write them down on wire

If `producer` is our Kafka configured producer (see on github for the details) all we need to do to send our message is:

```scala
producer.send(new ProducerRecord[String, RichEvent](topic, message))
```
where `topic` is of course the board we write on and `message` an instance of `RichEvent`, wrapped in a Kafka record. A great thing is that the `send` method is threadsafe, this is actually mentioned
in the JavaDoc actually, so you can have a single instance and have it used concurrently by multiple threads of course.

It is worth to have a look at the value serializer. At the end it will just be bytes ok, but a really convenient way to get there is to transform in JSON first; this also makes it easier to debug as
you see what you get, from the Kafka console tools too. For the purpose we will use [Argonaut](http://argonaut.io).

```scala
trait ArgonautSerializer[A] extends Serializer[A] {

  def encoder: EncodeJson[A]

  def serialize(topic: String, data: A) = encoder.encode(data).toString().getBytes

  def configure(configs: JMap[String, _], isKey: Boolean) = () 
  def close() = () // nothing to close

}
```  
`serialize` is the `Serializer` Kafka trait contract - make the value of type `A` a JSON string and binarize it, in our case. We don't configure anything, we could expect the charset but let's just
use the system default in the encode above. Also nothing to close as our serializer trait will be stateless - you should make sure you close internal resources here.

The implementation for our domain object will be:

```scala
class RichEventSerializer extends ArgonautSerializer[RichEvent] with Codecs {
  def encoder = richEventCodec.Encoder
}
```

Where in `Codecs` we have a `richEventCodec` that is just an Argonaut casecodec, very convenient.
Things are pretty much the same for the decoding part, where the contract method is:

```scala
def fromBytes(data: Array[Byte]) = Parse.decode(new String(data))(decoder).getOrElse { throw new ParseException(s"Invalid JSON: ${new String(data)}", 0) }
```

Ok, so to the consumer now.
The consumer implemented here is a direct Spark consumer, defined with just one line; it models a stream of Spark RDDs filled with the Kafka messages. Most importantly, in the direct consumer
the mapping between the number of topic partitions in Kafka and the partitions per Spark RDD is one-to-one, which is pretty cool as it bridges the two parallelism leverages in the libraries. Another
important point is that with a direct bridge an *at-most-once* delivery guarantee is offered. How is it configured?

```scala
ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG       -> "kafkabroker:9092",
ConsumerConfig.GROUP_ID_CONFIG                -> "mygroup",
ConsumerConfig.CLIENT_ID_CONFIG               -> "consumerid",
ConsumerConfig.ENABLE_AUTO_COMMIT_CONFIG      -> "true",
ConsumerConfig.AUTO_COMMIT_INTERVAL_MS_CONFIG -> "10000",
ConsumerConfig.AUTO_OFFSET_RESET_CONFIG       -> "smallest"
```

 * `BOOTSTRAP_SERVERS_CONFIG`: see above, this is needed as the direct receiver explicitly deals with the Kafka metadata.
 * `GROUP_ID_CONFIG`: important configuration for the consumer, as explained above 
 * `ENABLE_AUTO_COMMIT_CONFIG`: confirm to Kafka for the messages read so that it can move the offset on; important as the offset determines the next message that will be served
 * `AUTO_COMMIT_INTERVAL_MS_CONFIG`: how often to commit for the read messages
 * `AUTO_OFFSET_RESET_CONFIG`: defines the behavior when an unseen consumer first connects (see above)

The consumer stream is pretty easy to setup:

```scala
val consumer = KafkaUtils.createDirectStream[String, RichEvent, StringDecoder, RichEventDeserializer](streamingContext, consProps, Set(topic))
```

where `streamingContext` is of type `StreamingContext`, `consProps` are the consumer properties above.

Once your stream is setup you can register operations on it; if no output operation is registered on the stream Spark will complain there is nothing to do; you can try with `print()` or `count()`
or access the RDDs directly with `foreachRDD`. You can also associate a state to your stream by using `updateStateByKey`. In general, make sure you don't pull in classes and traits via their members
on you'll get infamous serialization errors, but this applies to Spark in general, not only its streaming tools.


If your consumer application is modeled as an [Akka](http://akka.io) actor then `streamingContext.start()` and `streamingContext.stop()` should be in your actor's `preStart()` and `postStop()` methods
and if you have instead a simple main you may want to use `streamingContext.start()` followed by `streamingContext.awaitTermination()`.

Empty queue, that's all!

[`git checkout`](https://github.com/rvvincelli/kafkasparkexample) the code!
