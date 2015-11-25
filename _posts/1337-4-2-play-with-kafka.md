##Play! with Kafka

When it comes to the web application backend in Scala and Java the [Play Framework](https://www.playframework.com) is for sure an important option to look at. Among its strengths, incorporated async
request serving, seamless JSON integration, powerful abstractions such as the iterator-iteratee model and websockets support. And of course it's [Typesafe](https://typesafe.com). A common use case
for a web service is to feed to client data coming from a continuous stream, especially now that WebSockets is a protocol - see [here](https://www.websocket.org). In Play! we have two ways to
implement the engine serving a websocket channel: actors or iteratees. Whereas it is IMHO easier to reason about and implement complex computations with actors, iteratees are the functional solution
and are a good fit if the data source we stream out requires little or no extra processing. Both options are documented [here](https://www.playframework.com/documentation/2.4.x/ScalaWebSockets).

Let's say that we want to integrate now Play! with [Apache Kafka](http://kafka.apache.org) (*integration*, what a word), simply meaning that we want to feed a websocket client the messages posted on
some Kafka topic. As after all a Kafka [high level consumer](http://kafka.apache.org/documentation.html#highlevelconsumerapi) gives us back a partition stream just as a list of pairs then it is of
course possible to massage it all into an iteratee and elegantly set up the stream. Let's say though that we want to set a delay in between consecutive events: this could be useful to simulate
deployment or production scenarios, or to fine tune, together with the use of batching, how many messages per time unit the client is able to handle and display without a performance degradation. For
this and similar ideas I find it easier to approach the problem using actors. Thanks to the infrastructure offered by Play! all we need to do is to create the actor feeding the websocket, no need to
care about anything else at protocol or socket level, awesome.

Let's start with defining our routes file in `conf/routes.conf`:

```
GET /kafka/:topics         io.github.rvvincelli.blogpost.playwithkafka.controllers.PlayWithKafkaConsumerApp.streamKafkaTopic(topics: String, batchSize: Int ?= 1, fromFirstMessage: Boolean ?= false)
```

So a GET where the argument is a list of topics, comma or colon separated for example, which supports a batch size and indicates whether to fetch all of the messages posted on the topic or only those
incoming after we start to listen.
We implement now our controller:
```scala
object PlayWithKafkaConsumerApp extends Controller with KafkaConsumerParams {

  private def withKafkaConf[A](groupId: String, clientId: String, fromFirstMessage: Boolean)(f: Properties => A): A = {
    val offsetPolicy = if (fromFirstMessage) "smallest" else "largest"
    f { consumerProperties(groupId = groupId, clientId = clientId, offsetPolicy = offsetPolicy) }
  }

  def streamKafkaTopic(topics: String, batchSize: Int, fromFirstMessage: Boolean) = WebSocket.acceptWithActor[String, JsValue] { _ => out =>
    Props {
      val id = UUID.randomUUID().toString
      withKafkaConf(s"wsgroup-$id", s"wsclient-$id", fromFirstMessage) {
        new PlayWithKafkaConsumer(_, topics.split(',').toSet, out, batchSize)
      }
    }
  }
  
}
```

`Controller` is just the Play! trait offering the standard ways to handle requests, thus `Action`s generating `Result`s. KafkaConsumerParams is a configuration trait for a Kafka consumer we have
introduced in the [previous](previous post). The `withKafkaConf` is a utility method to pass along the configuration parameters for the actor service `PlayWithKafkaConsumer`; these configuration
parameters define the Kafka messaging behavior. In particular:
* `topics`: a list of comma-separated topic names - we can set up streams for multiple topics in the same line
* `batchSize`: how many single Kafka messages we pack together and send to the client
* `fromFirstMessage`: for the client, whether to fetch all of the messages on the topic, or just retrieve those coming in after its connection  
See below for a few more points on these configuration properties.

And now to where the action is, `PlayWithKafkaConsumer`. Most importantly, let's setup the stream. To do this we first create a high-level consumer:
```scala
val consumer = Consumer.create(new ConsumerConfig(consumerSettings))
```
and configure the topic consumption:
```scala
val topicThreads = topics.map { _ -> 1 }.toMap[String, Int]
```
where `1` defines the number of streams we request for the topic. Under the hood this defines the number of threads will be employed by the consumer, and is not related to the number of partitions on
the topic; see the comments to [this](http://ingest.tips/2014/10/12/kafka-high-level-consumer-frequently-missing-pieces/) post for illuminating details. The takeaway is that this is not the number
of topic partitions and, no matter how many threads we attach to a partition, Kafka will automatically serve all of them always respecting the rule that a partition may be served to at most one single
consumer for a given group - at least as of version 0.8.1.1. As a reminder, the number of partitions per topic is a cluster-side configuration property.

The next step will be to actually create the streams:
```scala
val streams: List[(String, List[KafkaStream[String, RichEvent]])] = consumer.createMessageStreams[String, RichEvent](topicThreads, new StringDecoder(), new RichEventDeserializer()).toList
```




And now to the core:

```scala
stream.iterator().grouped(batchSize).foreach { mnms =>
    lazy val events = mnms.map { _.message() }
    client ! JsArray { events.map { event => Json.parse(reSer.encoder.encode(event).toString) } }
}
```

We first get a familiar iterator on the specific infinite stream implementation by Kafka. Then, we group it by the desired batch size, so that we automatically get a sequence of messages to send back
to the client. Very important point: if there are fewer messages than specified with the batch size this code will block! Might sound obvious, but it isn't, especially in situations like: batch size
is 10 and `kafka-consumer-offset` tells me there are 397 messages on the topic, I really can't get why the last 7 don't show up on the client. Anyways, it is generally better to send back a few huge
messages rather than flood the websocket client.
Finally, if we fetch from offset-zero or from now on only is decided with the `fromFirstMessage` switch, which just translates into an offset management value equal to `largest` or `smallest`,
new messages only versus all messages, respectively. The behavior should be decided according to what the application does, and left to the client, as you can see in the routes. If the web client
operating the socket is a browser the most flexible combination is to assign a random dynamic Kafka group ID, so that no messages may be stolen across tabs, and let the client set the boolean value.

