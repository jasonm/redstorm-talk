# RedStorm
## Distributed Realtime Computation in Ruby
# Jason Morrison
# December 11, 2012

---

#

(This is an HTML slide deck. Press "h" for help, or use the arrow keys to navigate. Press "p" for presenter notes, where you'll find a bunch of links, especially towards the end.)

---

# Batch vs Streaming

Design considerations

* Data retention
* Time scope of queries
    * TODO how to better phrase this ^
* Value of low latency

# Presenter Notes

  Hadoop and map-reduce are great for processing data in a batch mode, but that's not always a great fit.

  Instead of storing data and then executing batch queries over it,
  what if we persisted the query and ran the data through it?

  If you need to store all of your data and you need to execute queries which
  span large time-frames, or you do not know the queries up front, then Hadoop is great.

  However, when:

  * low latency is valuable
  * you know queries/functions ahead of time
  * query domain is a subset (time-localized) of all data

  Then a stream processing model can allow you to get at your answers in a much faster, and cheaper way.

---

# Have you ever...?

TODO queue+worker diagram

# Presenter Notes

  Have you ever written a queue+worker system to process incoming information?
  ...with multiple layers?
  ...fault tolerance
  ...data processing guarantees (at-least-once, exactly-once semantics)
  ...horizontal scalability
  ...transactional semantics

---

# Complex Event Processing (CEP)

_or "Event Stream Processing" (ESP)_

* Store queries instead of data
* Process each event in real-time
* Emit results when some query criteria is met

# Presenter Notes

* Complex Event Processing (CEP)
* Event Stream Processing (ESP)


---

# Design goals of storm:

* Higher level abstraction over message passing
* Guaranteed data processing
* Horizontal scalability
* Fault tolerance
* Easy to use

# Presenter Notes

* Remove intermediate queues/message brokers
    they are complex moving parts
    they are slower than direct worker:worker communication

---

# Use cases

* Web crawler
* Web analytics
* Twitter trending topics

# Presenter Notes

* So you have something concrete in mind while I discuss the parts of Storm
    * crawler (URL spout -> fetch -> resolve -> database)
    * analytics (clicks -> URL resolution -> bot filtering -> hadoop, cassandra)
    * trending topics (tweet stream -> tag extractor -> running counts)

---

# Storm primitives

* Streams
* Spouts
* Bolts
* Topologies

---

# Streams

# Presenter Notes

  streams
    core abstraction of storm
    unbounded sequences of tuples
    tuples are named list of values
    dynamically typed, use any type by providing serializer

---

# Spouts

# Presenter Notes

  spouts
    sources of streams
    e.g. read from a kestrel/kafka queue
         connect to a streaming API

---

# Bolts

# Presenter Notes

  bolts
    process any # input streams
    produce any # output streams
    where computation happens
      functions, filters, aggregations, streaming joins, talk to DBs...

---

# Topologies

# Presenter Notes

* network of spouts and bolts, connected by streams
* describe parallelism and field groupings

---

# Physical layout

* Tasks
* Workers
* Parallelism
* Stream groupings

TODO topology image?  tasks image? workers image?

# Presenter Notes


physical parts:
  tasks
    spouts and bolts are inherently parallel
    logical view of a topology @ runtime
  worker nodes
    physical view of topology @ runtime
    has any # of worker processes (JVMs) running
      each worker process has any # of tasks
  workers communicate directly with one another by passing messages
  stream grouping
    since spouts and bolts are parallel, when a tuple is emitted on a stream, which tasks does that tuple go to?
    decides how to partition the stream of messages
    * shuffle grouping: pick a random task, evenly distribute
    * fields grouping: mod hashing on a subset of tuple fields (aim for consistent recipient)
    * all grouping: send to all tasks
    * global grouping: pick task with lowest id (all tuples go to same task)
    looking at a topology, each edge is annotated with a stream grouping
---

# RedStorm

    !ruby
    class WordCountTopology < SimpleTopology
      spout RandomSentenceSpout, :parallelism => 5

      bolt SplitSentenceBolt, :parallelism => 8 do
        source RandomSentenceSpout, :shuffle
      end

      bolt WordCountBolt, :parallelism => 12 do
        source SplitSentenceBolt, :fields => ["word"]
      end
    end

# Presenter Notes

* Write topologies and components in JRuby
* Storm is Java and Clojure
* RedStorm is JRuby

introduce RedStorm
  storm is written for the JVM (mix of clojure and java) with APIs for Java and Clojure
  there are several ways to integrate across languages (namely: JVM-native, thrift for topos, json-over-stdio shelling for spouts/bolts)
  show an example
    * word_count_topology
  discuss how the code is setting up the topology
  run it locally to see what it looks like

---

# RedStorm

    !ruby
    class RandomSentenceSpout < RedStorm::SimpleSpout
      output_fields :word

      on_init do
        @sentences = [
          "the cow jumped over the moon",
          "an apple a day keeps the doctor away",
          "four score and seven years ago",
          "snow white and the seven dwarfs",
          "i am at two with nature"
        ]
      end

      on_send do
        @sentences[rand(@sentences.length)]
      end
    end

---

# RedStorm

    !ruby
    class SplitSentenceBolt < RedStorm::SimpleBolt
      output_fields :word

      on_receive do |tuple|
        sentence = tuple.getString(0)
        sentence.split(' ').map { |w| [w] }
      end
    end

---

# RedStorm

    !ruby
    class WordCountBolt < RedStorm::SimpleBolt
      output_fields :word, :count

      on_init do
        @counts = Hash.new(0)
      end

      on_receive do |tuple|
        word = tuple.getString(0)
        @counts[word] += 1

        [word, @counts[word]]
      end
    end


---

# Moving parts

rename this

* Nimbus node
* ZooKeeper nodes
* Supervisor nodes

# Presenter Notes

* local mode vs cluster mode
* 3 sets of nodes
* nimbus node: master node, similar to hadoop jobtracker
    * upload computations here
    * launches and coordinates workers, &
    *   moves them around when worker nodes fail
    * (not HA yet, but nimbus failure doesn't affect workers, so low-pri.  HA is on roadmap.)
* zookeeper nodes:
    * cluster coordination
* supervisor nodes:
    * talk to nimbus via ZK to decide what to run on the machine
    * start/stop worker processes as dictated by nimbus
* implementation: JVM (java+clojure), 0mq, zookeeper, thrift
* deployment: storm-deploy to automate deploment and provisioning on ec2

---

# How processing is guaranteed

TODO picture of tuple tree

# Presenter Notes

* tuple tree
    * rooted as a spout emits a single tuple
    * as bolts emit new tuples, they are anchored to the input tuple
* ack
    * after a bolt finishes, it acks its input tuple
    * after a whole tree is acked, the root tuple is considered processed
* this provides at-least-once semantics
* a topology specifies a timeout for retry
* _ who receives acks and decides to replay?  is it nimbus?  some quorum agreement amongst workers? via ZK?

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

# Higher-level abstractions

* DRPC: Ad-hoc computations across your cluster
* Trident: state, transactions, exactly-once semantics
* Lambda architecture

# Presenter Notes

* drpc
    * built atop storm primitives
    * design distributed computations as topologies
    * some function you want to execute on-the-fly, across a cluster
    * drpc server acts as a spout, emits function invocations
* trident
    * also built atop storm primitives
    * higher-level processing abstractions
    _ what's the relationship here with state and also abstractions
    * joins/grouping/aggregates/functions/filters
    * stateful source/sink
    * exactly-once semantics with fault tolerance
* combining both realtime and batch in your architecture
    * it's not uncommon to have both realtime computation and offline/batch processing needs
    * consider a queue/pubsub source like Kafka that is designed to flow into both offline and streaming systems
    * "lambda architecture" http://www.manning.com/marz/
    * unified query language: how to abstract over both realtime and batch processing

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
    * In-depth discussion of Storm from April 2012
    * "Storm: Distributed and fault-tolerant realtime computation"

* [Runaway complexity in Big Data](http://www.infoq.com/presentations/Complexity-Big-Data)
    * Contemporary thoughts on how to structure data systems
    * "Common sources of complexity in data systems and a design for a fundamentally better data system"

---

# Resources: Book

* ["Big Data"](http://manning.com/marz/)
    * Early access book by Nathan Marz
    * "Principles and best practices of scalable realtime data systems"

---

# Resources: Other ESP/CEP resources

* [Wikipedia: Event Stream Processing](http://en.wikipedia.org/wiki/Event_stream_processing)
* [Event Stream Processor Matrix](http://blog.sematext.com/2011/09/26/event-stream-processor-matrix/)
* [Quora: Are there any open-source CEP tools?](http://www.quora.com/Complex-Event-Processing-CEP/Are-there-any-open-source-CEP-tools)
* [Ilya Grigorik's "StreamSQL: Event Processing with Esper"](http://www.igvita.com/2011/05/27/streamsql-event-processing-with-esper/)
