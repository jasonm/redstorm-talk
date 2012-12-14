# Storm & RedStorm
# Distributed Realtime Computation in Ruby
# Jason Morrison
# December 11, 2012

![Storm](../images/topology.png)

---

# Data processing systems

# Presenter Notes

* design of data processing systems
* interesting case: 1 processing machine is not sufficient
* use cases like web/engagement analytics, app logging and performance analytics logging, financial/ecommerce information, physical sensor data
* could be for many reasons: high ingest volume, expensive computation, or tight latency requirements
* even if your processing system isn't large, storm does a very good job of documenting and exposing clean primitives and abstractions that are at play in these systems, and are valuable to understand

---

# DFS + map-reduce

* Queries are functions over all data
* Queries typically executed in batch mode, offline
* Results are high-latency

# Presenter Notes

* One popular approach for large-scale data processing systems is to dump the
  incoming data into a distributed filesystem like HDFS, run offline map-reduce
  queries over it, and place the results into a data store so your apps can read
  from it.

* If you need to store all of your data and you need to execute queries which
  span large time-frames, or you don't know the queries up front, then batch
  mode is a great fit.

  However, there are plenty of use cases where these parameters don't quite fit.

---

# Design considerations

* Value of low latency
* Data retention needs
    * Adhoc vs known queries
    * Timeframe of queries

# Presenter Notes

  Consider, when:

  * low latency is valuable
  * you know queries ahead of time
  * query domain covers small time windows

  then a stream processing model can allow you to get at your answers in a much
  faster, and cheaper way.

* Instead of storing data and then executing batch queries over it, what if we
  persist the query and run the data through it?

---

# Queues & workers

![workers and queues](../images/workers-queues.png)

# Presenter Notes

* First, I want to examine a typical approach to assembling a realtime system:

Hands up, ever written a system made of queues+workers to process incoming data?

* architecture here is
    * an incoming stream of data, from a web frontend or work queue or subscription to streaming API
    * cluster of queues persisting that data as jobs
    * cluster of workers taking jobs off the queue, working them, maybe persisting out to some store,
      and then emitting their processing results forward into the next queue layer
    * then more workers, maybe more persistence

Then you'll know there are some challenges...

* data processing guarantees & fault tolerance
* queues impose latency between layers
    * when one worker sends to another, impose a 3rd party between worker layers, where message has to go to disk
* without another layer of abstraction, coding is tedius - spending your time thinking about where to write message to, where do I read messages from, how do I serialize messages, etc.
* this is the kind of system that was in place in the product (Backtype) that became Twitter analytics

---

# Twitter analytics
![twitter analytics](../images/twitter-analytics.png)

---
# Storm
![storm on github](../images/storm-github.png)

# Presenter Notes

* released september 2011 at StrangeLoop
* >4,000 stars, >500 forks.  most starred Java project on github.
* Used by: Groupon, The Weather Channel, Alibaba, Taobao (ebay-alike, Alexa #11)

---

# Design goals.  Storm should:

* Guarantee data processing
* Provide for horizontal scalability
* Be fault tolerant
* Be easy to use

# Presenter Notes

* Guaranteed data processing: Choose whether, in the face of failure, data can be lost, or must be processed at-least once, or exactly once
* Horizontal scalability: Distribute your processing across a cluster, tweak parallelization of computations and the allocation of cluster resources to match your workload
* Fault tolerance: If there are any errors or when nodes fail, system should handle this gracefully and keep running
* Easy to use: As the application developer, focus on writing your computation, not infrastructure like message serialization, routing, and retrying.
* So, how is it put together?

---

# Storm primitives

* Streams
* _Components_
    * Spouts
    * Bolts
* Topologies

---

# Stream

![streams](../images/streams.png)

# Presenter Notes

* core abstraction of storm
* unbounded sequences of tuples
* tuples are named list of values
* dynamically typed, use any type by providing serializer
* streams are how that the parts of your computation talk to one another, they are the message passing

---

# Spout

![spouts](../images/spouts.png)

# Presenter Notes

* sources of streams
* typically:
    * read from a pubsub/kestrel/kafka queue
    * connect to a streaming API
* emits any number of output streams

---

# Bolt

![bolts](../images/bolts.png)

# Presenter Notes

* process any # input streams
* produce any # output streams
* where computation happens
    * functions, filters, aggregations, streaming joins, talk to DBs...

---

# Topology

![topology](../images/topology.png)

# Presenter Notes

* network of spouts and bolts, connected by streams
* describe parallelism and field groupings

---

# Parallelism: _logical view_

![parallelism](../images/parallelism.png)

# Presenter Notes

* nothing to do with hardware
* 4 components, 1 spout, 3 bolts.  #/tasks per.

---

# Parallelism: _physical view_

![workers](../images/workers.png)

# Presenter Notes

* 3 worker nodes
* multiple worker processes on each node (JVM process)
* tasks are threads, many tasks per thread

---

# Stream grouping

![streamgrouping](../images/streamgrouping.png)

* Shuffle grouping
* Fields grouping
* All grouping

# Presenter Notes

* since spouts and bolts are parallel, when a tuple is emitted on a stream, which tasks does that tuple go to?
* decides how to partition the stream of messages
    * shuffle grouping: pick a random task, evenly distribute
    * all grouping: send to all tasks
    * global grouping: pick task with lowest id (all tuples go to same task)
    * fields grouping: for persistent/stateful algorithms, send similar data to the same worker process.
      mod hashing on a subset of tuple fields

* let's take a look at some code...

---

# RedStorm

![RedStorm on github](../images/redstorm-github.png)


# Presenter Notes

* Storm is written for the JVM in Java and Clojure
* there are several ways to integrate across languages (namely: JVM-native, thrift for topos, "multilang components" json-over-stdio shelling for spouts/bolts)
* RedStorm lets you write topologies and components in JRuby

---

# Example time!

![word count topology](../images/word-count-topology.png)

# Presenter Notes

* RandomSentenceSpout
* SplitSentenceBolt
* WordCountBolt

---

# Data processing guarantees

Tuple tree:

![tuple tree](../images/tuple-tree.png)

# Presenter Notes

* now that we've seen...
* tuple tree
    * rooted at the spout, with a single tuple
    * as bolts emit new tuples, they are anchored to the input tuple
* ack
    * after a bolt finishes, it acks its input tuple
    * after a whole tree is acked, the root tuple is considered processed
* this provides at-least-once semantics
* you can build exactly-once processing semantics on top using transactions
---

# Cluster deployment

![storm cluster](../images/storm-cluster.png)

# Presenter Notes

* local mode vs cluster mode
* 3 sets of nodes
* Nimbus node: master node, similar to hadoop jobtracker
    * upload computations here
    * launches and coordinates workers, &
    *   moves them around when worker nodes fail
    * (not HA yet, but nimbus failure doesn't affect workers, so low-pri.  HA is on roadmap.)
* ZooKeeper nodes:
    * separate apache project
    * cluster coordination
        * store configuration, serve as distributed lock service for master election, etc.
* Supervisor nodes:
    * talk to nimbus via ZK to decide what to run on the machine
    * start/stop worker processes as necessary, as dictated by nimbus

* visibility: storm-ui


---

# Storm UI

![storm ui](../images/storm-ui.png)

# Presenter Notes

* what topologies are running on your cluster
* error logs
* details statistics about each topology
    * for every spout tuple, what's the avg latency until whole tuple tree is completed
    * for every bolt, avg processing latency
    * for every component, throughput

* deployment: storm-deploy to automate deploment and provisioning on ec2

---

# storm-deploy

    !yaml
    # /path/to/storm-deploy/conf/clusters.yaml

    nimbus.image: "us-east-1/ami-08f40561"      # 64-bit ubuntu
    nimbus.hardware: "m1.large"

    supervisor.count: 2
    supervisor.image: "us-east-1/ami-08f40561"  # 64-bit ubuntu
    supervisor.hardware: "c1.xlarge"
    #supervisor.spot.price: 1.60

    zookeeper.count: 1
    zookeeper.image: "us-east-1/ami-08f40561"   # 64-bit ubuntu
    zookeeper.hardware: "m1.large"

# Presenter Notes

* 1-click deploy tool for deploying clusters on AWS
* configure your cluster
* provides nice things like
    * spot pricing
    * ganglia for resource usage monitoring
* then configure your AWS settings

---

# storm-deploy

    !clojure
    ; ~/.pallet/config.clj

    (defpallet
      :services
      {
       :default {
           :blobstore-provider "aws-s3"
           :provider "aws-ec2"
           :environment {:user {:username "storm"
                                :private-key-path "~/.ec2/k.pem"
                                :public-key-path "~/.ec2/k.pem.pub"}
                         :aws-user-id "1234-5678-9999"}
           :identity "AKIA1111111111111111"
           :credential "abCDEFghijklmnpOPQRSTuvWXyz1234567890123"
           :jclouds.regions "us-east-1"
           }
        })

---

# storm-deploy

    !bash
    # start cluster
    $ lein run :deploy --start --name mycluster --release 0.8.1

    # attach to the cluster
    $ lein run :deploy --attach --name mycluster

    # stop cluster
    $ lein run :deploy --stop --name mycluster

---

# Running in production

    !bash
    # deploy your topology
    $ redstorm jar examples/simple
    $ redstorm cluster --1.9 examples/simple/word_count_topology.rb 

    # monitor with storm-ui and ganglia
    $ open http://{nimbus-ip}:8080
    $ open http://{nimbus-ip}/ganglia

    # kill topology
    $ storm kill word_count

# Presenter Notes

* jar it up, then submit the jar to the topology
* monitor with storm-ui and ganglia
* Specifying other dependencies for deploy (ami, system packages, jars, gems)
    * jclouds/pallet, apache ivy, bundler


---

# Bigger example: Tweitgeist

* [Live example](http://tweitgeist.colinsurprenant.com)
* [GitHub source](https://github.com/colinsurprenant/tweitgeist)

![tweitgeist](../images/tweitgeist.png)

---

# Bigger example: Tweitgeist

* [Talk: "Twitter Big Data"](http://www.slideshare.net/colinsurprenant/twitter-big-data)

![tweitgeist topology](../images/tweitgeist-topology.png)

---

# But wait, there's more...

* Runtime rebalancing and swapping
* DRPC: Ad-hoc computations
* Trident: state, transactions, exactly-once semantics
* Lambda architecture: blend streaming and batch modes

# Presenter Notes

* change parallelization and migrate topologies on the fly
* drpc
    * abstraction built on top of storm primitives
    * design distributed computations as topologies
    * some function you want to execute on-the-fly, across a cluster
    * drpc server acts as a spout, emits function invocations
* trident
    * also built atop storm primitives
    * exactly-once semantics with fault tolerance
    * stateful source/sink
    * higher-level processing abstractions
    * joins/grouping/aggregates/functions/filters
    _ what's the relationship here with state and also abstractions
* lambda
    * not part of storm,
    * but an approach to combining both realtime and batch in your architecture
    * discussed in 2012 strange loop talk
    * and in book "big data" http://www.manning.com/marz/

---

# Questions!

---

# Resources: Getting started

* [storm-starter](https://github.com/nathanmarz/storm-starter)
* [redstorm examples](https://github.com/colinsurprenant/redstorm/tree/master/examples)

---

# Resources: Software

* [RedStorm](https://github.com/colinsurprenant/redstorm)
* [storm](http://storm-project.net/)
* [storm-contrib](https://github.com/nathanmarz/storm-contrib)
* [storm-deploy](https://github.com/nathanmarz/storm-deploy)
* [storm-mesos](https://github.com/nathanmarz/storm-mesos)
* [storm-starter](https://github.com/nathanmarz/storm-starter)

---

# Resources: Documentation
* [Storm wiki](https://github.com/nathanmarz/storm/wiki)
    * ~ 40,000 words of doc
* [storm-user Google group](https://groups.google.com/group/storm-user)

---

# Resources: Talks

* [ETE 2012 Talk](http://vimeo.com/40972420)
    * "Storm: Distributed and fault-tolerant realtime computation" - April 2012

* [Runaway complexity in Big Data](http://www.infoq.com/presentations/Complexity-Big-Data)
    * "Common sources of complexity in data systems and a design for a fundamentally better data system" - October 2012

---

# Resources: Book

* [Big Data](http://manning.com/marz/)
    * Early access book by Nathan Marz
    * "Principles and best practices of scalable realtime data systems"

![big data book](../images/marz_cover150.jpg)

---

# Resources: Other ESP/CEP resources

* [Wikipedia: Event Stream Processing](http://en.wikipedia.org/wiki/Event_stream_processing)
* [Event Stream Processor Matrix](http://blog.sematext.com/2011/09/26/event-stream-processor-matrix/)
* [Quora: Are there any open-source CEP tools?](http://www.quora.com/Complex-Event-Processing-CEP/Are-there-any-open-source-CEP-tools)
* [Ilya Grigorik's "StreamSQL: Event Processing with Esper"](http://www.igvita.com/2011/05/27/streamsql-event-processing-with-esper/)

---

# Thanks!

[http://github.com/jasonm/redstorm-talk](http://github.com/jasonm/redstorm-talk)
* Diagrams from Nathan Marz' [Storm: Distributed and fault-tolerant realtime computation](http://chariotsolutions.com/presentations/storm-distributed-and-fault-tolerant-realtime-comp)
