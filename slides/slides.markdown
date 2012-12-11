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
* could be for many reasons: high ingest volume, expensive computation, or tight latency requirements

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

TODO remake image
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

* data processing guarantees
* fault tolerance
* queues impose latency between layers
    * with any reliability guarantees, they are complex moving parts
    * impose a 3rd party between worker layers, where message has to go to disk
* without another layer of abstraction, coding is tedius - spending your time thinking about where to write message to, where do I read messages from, how do I serialize messages, etc.

---
# Storm
![storm on github](../images/storm-github.png)

---
# Storm
![twitter analytics](../images/twitter-analytics.png)

---

# Design goals of Storm:

* Guaranteed data processing
* Horizontal scalability
* Fault tolerance
* No intermediate message brokers
* Easy to use

# Presenter Notes

* Guaranteed data processing: When errors happen, guarantee that data flowing into your cluster will be processed at least once, or exactly once, as you choose
* Horizontal scalability: Tweak parallelization of computations and cluster resources to match your workload
* Fault tolerance: If there are any errors or when nodes fail, system should handle this gracefully and keep running
* Remove intermediate queues/message brokers
    but without them, how do we handle fault tolerance?
* Easy to use: Focus on your computations, not infrastructure

---

# Example use cases

* Web crawler
* Web analytics
* Twitter trending topics

# Presenter Notes

* So you have something concrete in mind while I discuss the parts of Storm
    * crawler (URL spout -> fetch -> resolve -> database)
    * analytics (clicks -> URL resolution -> bot filtering -> hadoop, cassandra)
    * trending topics (tweet stream -> tag extractor -> running counts)
    * not going to cover this today, but the idea is to take a computation, implement it on storm, then let storm parallelize it, and invoke via storm-drpc

---

# Storm primitives

* Streams
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

---

# Spout

![spouts](../images/spouts.png)

# Presenter Notes

* sources of streams
* typically:
    * read from a pubsub/kestrel/kafka queue
    * connect to a streaming API

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

* 4 worker nodes, 1 spout, 3 bolts.  #/tasks per.

---

# Parallelism: _physical view_

![workers](../images/workers.png)

# Presenter Notes

* 3 worker nodes
* multiple worker processes on each node (JVM process)
* many tasks per process

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
    * fields grouping: mod hashing on a subset of tuple fields (aim for consistent recipient)
    * all grouping: send to all tasks
    * global grouping: pick task with lowest id (all tuples go to same task)

---

# RedStorm

![RedStorm on github](../images/redstorm-github.png)


# Presenter Notes

* Write topologies and components in JRuby
* Storm is written for the JVM in Java and Clojure
* there are several ways to integrate across languages (namely: JVM-native, thrift for topos, json-over-stdio shelling for spouts/bolts)
* RedStorm is a JRuby adapter library

---

# Example time!

---

# Word count topology

TODO resize image or something

![word count topology](../images/word-count-topology.png)

---

# Data processing guarantees

![tuple tree](../images/tuple-tree.png)

# Presenter Notes

* tuple tree
    * rooted as a spout emits a single tuple
    * as bolts emit new tuples, they are anchored to the input tuple
* ack
    * after a bolt finishes, it acks its input tuple
    * after a whole tree is acked, the root tuple is considered processed
* this provides at-least-once semantics
* a topology specifies a timeout for retry TOPOLOGY_MESSAGE_TIMEOUT_SECS default 30s
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
    TODO improve this description from wiki
* Supervisor nodes:
    * talk to nimbus via ZK to decide what to run on the machine
    * start/stop worker processes as necessary, as dictated by nimbus
* implementation: JVM (java+clojure), 0mq, zookeeper, thrift
* deployment: storm-deploy to automate deploment and provisioning on ec2


---

# Storm UI

TODO: show the screen

# Presenter Notes

  ete@45:15
  what topos are running on your cluster
  details statistics about each cluster
  - for every spout tuple, what's the avg latency until whole tuple tree is completed
  - for every bolt, avg processing latency
  - for every component, throughput

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
* show off running on ec2

---

# storm-deploy

    !bash
    # start a cluster
    $ lein run :deploy --start --name mycluster --release 0.8.1

    # stop a cluster
    $ lein run :deploy --stop --name mycluster

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

# Tweitgeist

[https://github.com/colinsurprenant/tweitgeist](https://github.com/colinsurprenant/tweitgeist)
[http://tweitgeist.colinsurprenant.com](http://tweitgeist.colinsurprenant.com)

![tweitgeist](../images/tweitgeist.png)

---

# Higher-level abstractions

* DRPC: Ad-hoc computations
* Trident: state, transactions, exactly-once semantics
* Lambda architecture: blend streaming and batch modes

# Presenter Notes

* drpc
    * built atop storm primitives
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

![big data book](http://manning.com/marz/marz_cover150.jpg)

---

# Resources: Other ESP/CEP resources

* [Wikipedia: Event Stream Processing](http://en.wikipedia.org/wiki/Event_stream_processing)
* [Event Stream Processor Matrix](http://blog.sematext.com/2011/09/26/event-stream-processor-matrix/)
* [Quora: Are there any open-source CEP tools?](http://www.quora.com/Complex-Event-Processing-CEP/Are-there-any-open-source-CEP-tools)
* [Ilya Grigorik's "StreamSQL: Event Processing with Esper"](http://www.igvita.com/2011/05/27/streamsql-event-processing-with-esper/)

---

# Thanks!

[http://github.com/jasonm/redstorm-talk](http://github.com/jasonm/redstorm-talk)





...........................................
CLIP
...........................................

---

# Complex event processors (CEP)

_or "Event stream processors" (ESP)_

* Store queries instead of data
* Process each event in real-time
* Emit results when some query criteria is met

# Presenter Notes

* Family of tools that work just like that
* Varying degrees of abstraction
* Storm is one, there are more, ... Esper, S4, Drools Fusion, FlumeBase

