The past decade has seen a revolution in data processing. MapReduce, Hadoop,
and related technologies have made it possible to store and process data at
scales previously unthinkable. Unfortunately, these data processing
technologies are not realtime systems, nor are they meant to be. There's no
hack that will turn Hadoop into a realtime system; realtime data processing has
a fundamentally different set of requirements than batch processing.

> Technologies like MapReduce and Hadoop enable us to store and process data at scale.
> But these are not realtime systems.

Ilya Grigorik, writing about Esper:

> Instead of storing massive data and executing batch queries over it, what if we persisted the queries instead and ran the data through as it arrives?
> "Event Stream Processing" systems do just this.

> Batch processing systems like Hadoop fit well when you need to store all the data and execute queries that span large timeframes,
  and when you do not know the queries up-front.

  Batch systems make idempotence/exactly-once semantics easy.

  The disadvantage is the high latency of queries and the need to store the data.

> Stream processing systems like Esper and Storm fit well primarily when low latency is valuable.
  when low latency is valuable.

$   One important thing to recognize is that 
$   there are fundamental differences between batch and realtime 
$   computation. Echoing what Ashley said, with batch processing it's easy 
$   to get idempotence, whereas when dealing with incremental algorithms 
$   and at-least-once semantics it's harder to achieve idempotence. 

$   ^^ example: counting.  at-least-once + incremental -> idempotence is hard.
$      you'd need exactly-once semantics; otherwise counts would be too high.

$   ^^ so, in storm, how would you reliably and accurately (exactly-once)
$      count something?

$      read: Transactional-topologies
$            Trident-tutorial
$            Trident-state

$   The differing semantics between batch and stream processing means that 
$   the tools for batch processing don't have a clear mapping to stream 
$   processing. For example, joins as defined in batch processing don't 
$   make sense when dealing with infinite data streams. 

$   Additionally, you don't always do the same thing in batch as in 
$   realtime. It's common to use an approximation algorithm for the 
$   realtime portion and the exact algorithm for the batch portion, and 
$   the batch portion corrects the realtime portion over time. 

> If you need to store all of your data and you need to execute queries which
  span large time-frames, or you do not know the queries up front, then Hadoop is
  great. However, in many cases, a stream processing model can allow you to get
  at your answers in a much faster, and cheaper way. It's not an either-or,
  rather, I feel like stream processing is just an under-explored area in
  general. In fact, you're probably best of using both systems in the right
  contexts.

However, realtime data processing at massive scale is becoming more and more of
a requirement for businesses. The lack of a "Hadoop of realtime" has become the
biggest hole in the data processing ecosystem.

Storm fills that hole.

Before Storm, you would typically have to manually build a network of queues
and workers to do realtime processing. Workers would process messages off a
queue, update databases, and send new messages to other queues for further
processing. Unfortunately, this approach has serious limitations:

1. **Tedious**: You spend most of your development time configuring where to
   send messages, deploying workers, and deploying intermediate queues. The
realtime processing logic that you care about corresponds to a relatively small
percentage of your codebase.
2. **Brittle**: There's little fault-tolerance. You're responsible for keeping
   each worker and queue up.
3. **Painful to scale**: When the message throughput get too high for a single
   worker or queue, you need to partition how the data is spread around. You
need to reconfigure the other workers to know the new locations to send
messages. This introduces moving parts and new pieces that can fail.

Although the queues and workers paradigm breaks down for large numbers of
messages, message processing is clearly the fundamental paradigm for realtime
computation. The question is: how do you do it in a way that doesn't lose data,
scales to huge volumes of messages, and is dead-simple to use and operate?

Storm satisfies these goals. 

## Why Storm is important

Storm exposes a set of primitives for doing realtime computation. Like how
MapReduce greatly eases the writing of parallel batch processing, Storm's
primitives greatly ease the writing of parallel realtime computation.

The key properties of Storm are:

1. **Extremely broad set of use cases**: Storm can be used for processing
   messages and updating databases (stream processing), doing a continuous
query on data streams and streaming the results into clients (continuous
computation), parallelizing an intense query like a search query on the fly
(distributed RPC), and more. Storm's small set of primitives satisfy a stunning
number of use cases.
2. **Scalable**: Storm scales to massive numbers of messages per second. To
   scale a topology, all you have to do is add machines and increase the
parallelism settings of the topology. As an example of Storm's scale, one of
Storm's initial applications processed 1,000,000 messages per second on a 10
node cluster, including hundreds of database calls per second as part of the
topology. Storm's usage of Zookeeper for cluster coordination makes it scale to
much larger cluster sizes.
3. **Guarantees no data loss**: A realtime system must have strong guarantees
   about data being successfully processed. A system that drops data has a very
limited set of use cases. Storm guarantees that every message will be
processed, and this is in direct contrast with other systems like S4. 
4. **Extremely robust**: Unlike systems like Hadoop, which are notorious for
   being difficult to manage, Storm clusters just work. It is an explicit goal
of the Storm project to make the user experience of managing Storm clusters as
painless as possible.
5. **Fault-tolerant**: If there are faults during execution of your
   computation, Storm will reassign tasks as necessary. Storm makes sure that a
computation can run forever (or until you kill the computation).
6. **Programming language agnostic**: Robust and scalable realtime processing
   shouldn't be limited to a single platform. Storm topologies and processing
components can be defined in any language, making Storm accessible to nearly
anyone.
