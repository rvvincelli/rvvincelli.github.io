---
layout: post
title: Pump that gear
---

{% include google_analytics.html %}

##Everything is an actor

I have had the chance to play a little with [Gearpump](http://www.gearpump.io), an [Akka](http://akka.io)-based data processing framework. A few very interesting features are:

* stream processing: simply define event sources and sinks, together with all that should happen in between, with data processed in a standard streaming fashion
* clean scaling: the architecture could be seen as Spark-like but the roles and control flows are clearly defined as we talk about actors here 
* easy job composition: jobs are composed with a flow-based DSL, so if you are used to working with [Akka streams](http://doc.akka.io/docs/akka-stream-and-http-experimental/2.0.3/scala/stream-quickstart.html) for example it is all really easy
* modular connectors: a few source and sink connectors are provided already, and you are free to create custom ones targeting your Hadoop beast of choice
* hassle-free scaling: define your actor workers and configure how many of them you need

##What we are going to do

Let us go through a full example to see how things actually work, together with the deployment of a Gearpump program on YARN. We will prepare a modified version of the wordcount example [found](http://www.gearpump.io/releases/latest/dev-write-1st-app.html)
in the documentation. We will:

* read the text from a [Kafka](http://kafka.apache.org) queue - this is our source
* split it and count the words - our tasks
* store the counts per word in [HBase](https://hbase.apache.org)

Deploying the Gearpump YARN infrastructure and setting up our data source are part of the tutorial too.

##Source and sink setup

Our data source will be a Kafka topic containing the text to process and we will use an HBase table collecting the counts. We assume to work on a standard [Cloudera](https://cloudera.com/products/cloudera-manager.html)
cluster, not that it really matters, any distribution of your choice will do.

Let's load our data into Kafka:

`kafka-console-producer --broker-list broker:9092 --topic loremipsum --new-producer < randomipsum.txt`

so we pour the textfile into a new topic `loremipsum`; `--new-producer` uses the new Kafka producer implementation - without it the command failed, at least on my cluster - I have not investigated why.
Notice that in Kafka topics may always be created on-the-fly, ie as soon as the first message is posted. In that case they are created with default cluster settings, eg only one partition. To create the topic beforehand with proper partitioning and redundancy settings:

`kafka-topics --zookeeper quorum:2181 --partition 3 --replication-factor 3 --create --topic loremipsum`

then you can use the console producer to pour the data.

Finally we pour the data in HBase. Surprisingly, Gearump does not take care of creating the table for us if it does not exist, at least on the version I tested. In general, use the HBase shell:

`$hbase shell`
`>create randomipsum','counts'`

this creates a new table in the `default` namespace.

##The code

Our application is the Hadoop HelloWorld:

*read the data from a source
*split the data in batches for processing
*perform some operation on the single batches
*aggregate the results

In our particular case:

*read the data from a Kafka topic
*split it by line
*for every batch, count the words and sum it
*store the results

The application definition is a oneliner:

```scala
val app = StreamApplication("wordCount", Graph(kafkaSourceProcessor ~> split ~ partitioner ~> sum ~> hbaseSinkProcessor), UserConfig.empty)
```

This expression will be familiar if you worked with Akka streams or any other flow-oriented framework, what we actually build is a graph, an execution graph:
*pick up the data from Kafka
*split it
*route it to the summers according to a partitioning scheme
*store the result in HBase

Let's see how we build every computing entity. 

With a little taste for abstraction, we define a provider trait for our Kafka processor needs:

```scala
trait KafkaSourceProvider { self: KafkaConfProvider =>

  implicit def actorSystem: ActorSystem
    
  private lazy val zookeepers = s"$zookeeperHost:$zookeeperPort"
  private lazy val brokers = s"$brokerHost:$brokerPort"
		  
  private lazy val offsetStorageFactory = new KafkaStorageFactory(zookeepers, brokers)
			
  private lazy val kafkaSource = new KafkaSource("randomipsum", zookeepers, offsetStorageFactory)
			    
  protected lazy val kafkaSourceProcessor = DataSourceProcessor(kafkaSource, 1)
}
```

A few bits of Kafka-related configuration of course, coming from a configuration provider, and the creation of a Gearpump Kafka source. This is more Scala than anything, but notice how we must require for an Akka actor system in the contract.

The split agent is defined like this:

```scala
class Split(taskContext : TaskContext, conf: UserConfig) extends Task(taskContext, conf) {
  import taskContext.{output, self}
  
  override def onNext(msg : Message) : Unit = {
    new String(msg.msg.asInstanceOf[Array[Byte]]).lines.foreach { line =>
      line.split("[\\s]+").filter(_.nonEmpty).foreach { msg =>
        output(new Message(msg, System.currentTimeMillis()))
      }
  }
  
  import scala.concurrent.duration._
  taskContext.scheduleOnce(Duration(100, TimeUnit.MILLISECONDS))(self ! Message("continue", System.currentTimeMillis()))
}
```

What we define here is an actor task, whose instances are managed by the Executor Application Actor, the workhorse, which is a child of the Worker actor for that cluster node. Here sits the key to flexible parallelism and thus performance - one may spin up as many task actor instances as needed. You can read about Gearpump internals [here](http://www.gearpump.io/releases/latest/gearpump-internals.html).
See how the `Message` is actually containing the data, and yes it is really awful to beg `instanceOf` just to get a mere array of bytes - future releases may have considered powerful abstractions via scalisms like macros.
This computation is started once and continued forever - we just keep waiting for new messages to come and process them; this typical actor design pattern is explained [here](http://www.gearpump.io/releases/latest/dev-write-1st-app.html).
The task itself is nothing new, just remember we use a `foreach` since we have a void return, given that we are firing off the message chunk.

The next step is to route the text splits to the Applications for the sum. The easiest way is to hash-partition, on the usual uniformity assumption the workload will be split evenly.

And now the sum - notice that once you break out of the mapreduce paradigm you can just aggregate as much as you like, a Task can do anything. But unless the operations are really easy and fit into the listisms of Scala the best thing it to reason in terms of *one task equals one transformation/operation*:

```scala
class Sum (taskContext : TaskContext, conf: UserConfig) extends Task(taskContext, conf) {
  ...
  private[gearpump_wordcount_kafka_hbase] val map : mutable.HashMap[String, Long] = new mutable.HashMap[String, Long]()
  ...
  private var scheduler : Cancellable = null

  overrride def onStart(startTime : StartTime) : Unit = {
    scheduler = taskContext.schedule(new FiniteDuration(5, TimeUnit.SECONDS),
    new FiniteDuration(30, TimeUnit.SECONDS))(reportWordCount)
  }
  
  override def onNext(msg : Message) : Unit =
    if (null != msg) {
	  val current = map.getOrElse(msg.msg.asInstanceOf[String], 0L)
	  wordCount += 1
	  val update = (msg.msg.asInstanceOf[String], "counts", "count", s"${current + 1}")
	  map + ((update._1, update._4.toLong))
	  output(new Message(update, System.currentTimeMillis()))
	}
  }
  ...
}
```

Every string we get is a word. We keep an internal state for the counts, the `map` object. We increase the counter for the word, which will be non-zero if the word has been seen before. The outbound message has a rather complex and undocumented structure, it contains the key, family, column name and value.

How are the two actor types made known to Gearpump? Simple with:

```scala
val split = Processor[Split](1)
val sum = Processor[Sum](1)
```

specifying the number of task instances too.

Finally, the HBase sink. This is simply:

```scala
trait HBaseSinkProvider { self: KafkaConfProvider =>
  
    implicit def actorSystem: ActorSystem
	  
    private val principal = "rvvincelli@modoetia"
	private val file = Files.toByteArray(new File("/home/rvvincelli/rvvincelli.keytab"))
		    
	private val userConfig = UserConfig.empty
		.withString("gearpump.kerberos.principal", principal)
		.withBytes("gearpump.keytab.file", file)
					    
	private def hadoopConfig = {
		val conf = new Configuration()
		conf.set("hbase.zookeeper.quorum", zookeeperHost)
		conf.set("hbase.zookeeper.property.clientPort", zookeeperPort)
		conf
	}
											    
	private lazy val hbaseSink = HBaseSink(userConfig, "randomipsum", hadoopConfig)
												    
	protected lazy val hbaseSinkProcessor = DataSinkProcessor(hbaseSink, 1)
													  
}
```

We can see the Kerberos configuration, asking to read the two variables as a string and as a binary file respectively. The ZooKeeper configuration properties are mandatory, you might get funky errors without them.

As usual [here](https://github.com/rvvincelli/gearpump-wordcount-kafka-hbase) is the full code.

##How to deploy

We abstract from a particular Hadoop distribution, but a common important point is to support [Kerberos](https://en.wikipedia.org/wiki/Kerberos_(protocol)). In this respect, Gearpump comes with integrated Kerberos configuration, which is cool and makes a crucial checkbox for production.

First, download the Gearpump precompiled package [here](http://www.gearpump.io/downloads.html) and upload it to a suitable HDFS location, eg `/usr/lib` or your home.

Locally, unzip this package and copy the contents of the Hadoop configuration directory `/etc/hadoop` into the conf subdirectory.

Now launch the Gearpump cluster YARN application, eg: `bin/yarnclient launch -packa
ge /user/rvvincelli/gearpump-2.11-0.7.5.zip`; your application should appear under 
the Applications tab for YARN; if not, investigate the logs.

Get a copy of the active configuration:

`bin/yarnclient getconfig -appid <APPID> -output conf/mycluster.conf` 

where `<APPID>` is the application ID from the tab above; you can't use the configuration from previous Gearpump cluster runs, fetch it anew.

We are now ready to launch the application!

`bin/gear app -jar ../gearpump-wordcount-kafka-hbase-assembly-1.0.jar -conf conf/mycluster.conf`

and if you see `Submit application succeed` in the output then everything went fine :)

##<DEBUG INFO>

Mmake sure the application is running by connecting to the Gearpump webapp (see [here](http://www.gearpump.io/releases/latest/deployment-ui-authentication
.html)) - you find the link in the container logs of the Gearpump instance on the Y
arn Resource Manager webapp.
Once you log in click on the `Details` button for your application - if it is not enabled then the application has already terminated time ago. In the `Overview` tab click on `Log dir.` and move to this path on the box where the appmaster actor is running - you see this in the `Actor Path` entry.
 
To make sure the data is there fire up an HBase shell and scan the table:
 
`t = get_table 'randomipsum'`
`t.scan`

That's it, pump your gears!
