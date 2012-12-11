# RedStorm
# Distributed Realtime Computation in Ruby
# Jason Morrison
# December 11, 2012

![Storm](../images/topology.png)

---

# Data processing systems

# Presenter Notes

* 1 processing machine is not sufficient

---

# Design considerations

* Data retention needs
    * Adhoc vs known queries
    * Timeframe of queries
* Value of low latency

# Presenter Notes

* One popular approach for large-scale data processing systems is to dump the
  incoming data into a distributed filesystem like HDFS, run offline map-reduce
  queries over it, and place the results into a data store so your apps can read
  from it.

* If you need to store all of your data and you need to execute queries which
  span large time-frames, or you don't know the queries up front, then batch
  mode is a great fit.

  However, when:

  * low latency is valuable
  * you know queries ahead of time
  * query domain is a subset of all data across a relatively small time window

  then a stream processing model can allow you to get at your answers in a much
  faster, and cheaper way.

* Instead of storing data and then executing batch queries over it, what if we
  persist the query and run the data through it?

---

# Complex event processing (CEP)

_or "Event stream processing" (ESP)_

* Store queries instead of data
* Process each event in real-time
* Emit results when some query criteria is met

---

# Have you ever...?

![workers and queues](../images/workers-queues.png)

# Presenter Notes

  Have you ever written a queue+worker system to process incoming information?
  ...with multiple layers?
  ...fault tolerance
  ...data processing guarantees (at-least-once, exactly-once semantics)
  ...horizontal scalability
  ...transactional semantics

---

# Design goals of Storm:

* No intermediate message brokers
* Fault tolerance
* Horizontal scalability
* Easy to use

# Presenter Notes

* Fault tolerance: When errors happen, guarantee that data flowing into your cluster will be processed at least once, or exactly once, as you choose
* Remove intermediate queues/message brokers
    they are complex moving parts
    they are slower than direct worker:worker communication
    but without them, how do we handle fault tolerance?
* Horizontal scalability: Tweak parallelization of computations and cluster resources to match your workload
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

# How processing is guaranteed

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

1-click deploy tool for deploying clusters on AWS
show off running on ec2
TODO what are the commands

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
