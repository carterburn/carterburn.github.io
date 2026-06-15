---
title: "Gossip Glomers - Challenge 3d - 6"
layout: single
tags:
    - gossip_glomers
    - rust
    - distributed systems
---

Hello, welcome back! I'm here to round out the Gossip Glomers challenges in this
post. In the last post on Gossip Glomers, I was kicking off challenge 3d. Now,
I'll start this post and say that 3d and 3e were likely the most difficult
challenges in the set from an engineering standpoint. Later challenges forced me
to learn some different algorithms or approaches to solving problems, but 3d and
3e tested the engineering skills with pulling multiple levers to pass the
challenge. 

Beyond that, challenge 4 through 6 ended up forcing me to learn some new
concepts and give it a shot. I'll finish everything up here.

Overall, the challenges were great and allowed me to see a bit of a taste of
distributed systems engineering in real life. I wish there were more sets like
this out there because it truly is a great way to test your skills!

## Challenge 3d/3e
The point with 3d and 3e is to get the broadcast to hit certain measurement
components. First, 3d tries to get the median latency below 400ms and the max
latency under 600ms with a 30 msgs/op (30 messages exchanged on average for each
operation sent by Maelstrom) while 3e relaxes the latencies (1 and 2 seconds
respectively) while driving msgs/op to 20. I set out to try to hit both of these
targets in one binary: meaning we'd have 20 msgs/op with a 400ms median latency
and 600ms latency. One note, there is no more partition, but I still kept the
ACK infrastructure in place to still be able to pass 3c as well. The challenge
here was basically hitting all of these metrics within one binary (I stopped
doing this in subsequent challenges...). 

The biggest realization here was the difference between theory and measurement.
For theory, it appears that a grid topology is the best bet here. So, at init, I
sorted the node list lexicographically and then took: $\left\lceil \sqrt{n} \right\rceil$ 
where `n` is the number of nodes. What this provides is a grid like pattern
where each node will have a few nodes to broadcast values to. This theory allows
each node to get maximum reach in as few messages as possible. Once the
neighbors receive that value, the neighbors also sprawl out and the entire
set of nodes quickly receives a value. This helps reduce msgs/op as well as
latency to get messages spread as quick as possible while also (hopefully)
ensuring a partition doesn't cause too much delay (as long as there isn't a
complete partition where nodes are fully separated into two separate networks;
but the ACKs from 3c would come in handy). 

Another realization was to keep a `HashMap` of novel values for each neighbor
(`HashMap<String, HashSet<usize>>`). What this provided was the ability to track
a set of values the node has strong confidence are not known by the specific
neighbors. For example, on a new value from a client, every neighbor (all keys
in the `HashMap`) have the new value added to the `HashSet`. Then, anytime a
neighbor propagates a value to a node, that neighbor won't have its value added
to the `HashSet` because the node that received the propagate message doesn't
need to send that value back to the node that sent it. 

On a few runs, the times were looking really good. Latency was usually well
under, but the msgs/op stayed fairly high. With a little debugging help from
Claude, I realized the issue was that I was: 1) ACKing before the latency delay
and 2) I could batch a bit better with that cushion in the latency numbers. The
first step was to reduce that re-propagate with the ACK. I was sending values
that had no possible chance to reach their intended node with the Maelstrom
latency delay. I needed to give at least a roundtrip (and likely more) to let
the value have a chance to get to the intended node and the ACK back to the
sender. I added a tick tracker and ensured that at least 3 ticks went by with no
ACK before resending a value. That obviously helped drive the msgs/op down, but
it wasn't quite enough to pass the bar. The next move was to increase the tick
interval. Now, this sounds silly at first, but what it did was allowed more
messages to batch up, so I could move from sending small messages with a value or two 
in them to sending larger messages less frequently. I eventually landed on a tick
interval of 240ms and that cleared the msgs/op and latency bars for challenge
3d/3e. 

This challenge taught me two main things: 1) the theory can only take you so
far; sometimes you have to engineer your way to things (increase interval to
send messages in order to decrease overall latency) and 2) measuring where you
are at in a given solution is good to tell you how much more engineering work is
required. I had plans for an even more robust solution for the propagation, but
it turned out it was not needed! 

The final challenge 3 binary can be found [here](https://github.com/carterburn/Gossip-Glomers/tree/main/challenges/broadcast).

## Challenge 4
[Challenge 4](https://fly.io/dist-sys/4/) was my first real introduction to Conflict-free Replicated Data Type (CRDTs).
These data structures are pretty neat in my opinion. They allow multiple nodes
to make edits completely on their own and then merge together when a network
heals. Its how things like Figma allow multiple editors to make offline
edits merge seamlessly, so not a bad thing to learn. For challenge 4, the CRDT to implement
is the basic one, the Grow-Only Counter. This is 'basic' because all you do is
keep a local counter and then add another node's counter when there is a `merge`
operation. 

The trick here is that there are a lot of nodes to deal with, so the solution is
to use the `seq-kv` provided key-value store by Maelstrom. This gives sequential
consistency for reads and writes (more on that later). I used this key-value
store to have every node store their counter with a simple Write operation and
then other nodes can read that value when there is a Read operation. I could
have also used two other approaches: 

1. Compare-And-Swap a single counter, but I
did test this because I was curious and this didn't pan out because of the
sequential nature of `seq-kv`. There was a stale read at the end that caused the
workload to fail with this approach.
2. Completely drop `seq-kv` and gossip a counter every time a client asks to add a 
    value. This is probably the more
   'production' approach to a GCounter, but if `seq-kv` is provided, I figured I
   might as well use it. I didn't implement this, but it could be interesting to
   try it out.

The approach with just writing to a per-node key seemed to work well. In the
first design, I actually held off on ACK'ing the client's request until I
confirmed that the write to the key-value store succeeded. It turns out that
that was over-engineered (and even could have caused some issues). It didn't
need to be that complicated. All I did for a second approach was to
optimistically ACK the client's request and then on a tick interval, write the
current local total. One major benefit of this was that I then 'batched' adds
together. If two client requests came in during a single tick interval, both of
those requests would be reflected in the local total and required only one
message to the key-value store. It was still correct and it was more efficient.
Like to see that! 

Overall, this was a good challenge, but it did seem easier than challenge 3.
More complicated CRDTs would have been an additional challenge, but it was a
nice, gentle introduction to them. 

Full code for chapter 4 is [here](https://github.com/carterburn/Gossip-Glomers/tree/main/challenges/gcounter).

## Challenge 5
For [challenge 5](https://fly.io/dist-sys/5a/), it was time to create a
Kafka-like log. For this challenge, it wasn't necessarily hard to understand the
theory, but the architecture and design of the node was fairly involved.
Challenge 5 starts with a single node that accepts requests, so it was meant to
just get a feel for the types of messages and operations that needed to occur.
As a result, I just did everything in memory for the single node. The log was
stored as a `HashMap<String, Vec<usize>>`. It worked pretty much out of the box
besides an assumption that indices started at 1 for the workload when they
really start at 0. That simple fix got 5a completed.

For 5b, now there were multiple nodes. To handle this, I used the `lin-kv` store
provided by Maelstrom to store the log. It was no longer stored per node and
rather stored within the provided key-value store. To handle this, there were
three key flavors that were used in the key-value store provided:

1. `log/<key>/<offset>`: stored the actual value in a key's log at the specified
   offset
2. `head/<key>`: stored the next offset to provide for a new value added to the
   log
3. `committed/<key>`: stored the committed offset value for a given key

So, how did this look in practice? When a client sent a `Send` RPC to add a
value to a key's log, the node would first attempt to read the current head for
that key. Once that read came back, then the node would attempt a
Compare-and-Swap on the head for that key to the read value + 1. If the CAS
succeeded, the node would effectively 'own' that offset now and be able to
dispatch a simple write to the `log/<key>/<offset>` with its owned offset to
store the actual value. Once that write came back, then the node would ACK the
client that it successfully add that value to the log. That is critical so that
we can guarantee the correctness with the log. The node provides 'CP'
(consistent, partition tolerant) to the clients at this stage so it's okay to
block the client until the write comes back (note: there is no actual partition
happening and we could easily have provided 'CA' and we will in 5c but this is
not necessary to succeed in this challenge). 

For commit offsets, the idea is the same, the node will first read the current
committed offset and then send a CAS with the from value as the returned read
value and the to as the value the client requested to commit. The nice part
about this portion of the workload is that if the read comes back with an offset
greater than or equal to the offset that the client requested, the node can
immediately return to the client and not attempt the CAS.

For both of these portions, there was a good deal of machinery used to track
these operations through their completion. I started with a `PendingOp` `enum`
that tracked each `lin-kv` op through its various stages. For `Send`, there was
the read from the key-store, the CAS, and then the write. A good number of these
pending ops also had an `Aggregator` that would allow each operation type to
aggregate their results. For example, with a `Poll` RPC, the node needs to
submit a good number of requests to `lin-kv` (one for each offset in the poll at
`log/<key>/<offset>`). When each of those read's comes back, then the aggregator
will store it off and collect it at the end. 

Another key consideration was dealing with RPC errors. Certain error codes (like
the CAS contention or key not found) actually were built into the workflow. If
there was a key not found in the Poll operation, that meant the poll stopped
there and the client received their response. That error was almost treated like
Rust's `take` on `Iterator`'s in that it tries to take as many elements as
possible up to a maximum but will stop once its exhausted. For CAS contentions,
that typically meant that the node needed to reissue a Read request because
another node beat it to the punch in a concurrent CAS request for that key.

While implementing this Read->CAS->Write loop, I found a fairly interesting bug
that I thought was worth mentioning. When a client makes a `Send` RPC to add a
value to the log, the first operation, as mentioned above, is a read operation
for the `head/<key>` key in the `lin-kv` key-value store to attempt to claim the
next offset. If that read comes back with an error `KeyDoesNotExist`, that means
this is the first time a client is requesting to add a value to this log.
Naively, I then setup the CAS for that same key to claim offset 0 with a
`from=0,to=0`. I thought: "Well, the key hasn't been created yet, so the from
value doesn't really matter if I'm setting `create_if_not_exists` to true". That
assumption turned out to be very wrong in the case of concurrent clients trying
to write the first value to a log. What happened was two nodes sent the CAS
message; one of them landed first, meaning there was a successful response and
gave that node the indication that it was safe to claim index 0 (the `to` value
was 0). Then the second one was processed by `lin-kv` and *also* returned a
success message because the `from` value was 0 in that request and the key's
value was, in fact, 0 because of the successful CAS from the other node. What
this meant was that a good chunk of those first log messages were getting
clobbered in the log itself. Two nodes received a success response from CAS and
then wrote at `log/<key>/0`, meaning one of them would clobber the other. This
was definitely an interesting bug, but it made complete sense. To address it, I
simply used `from=usize::MAX` in situations where the initial read on the
`head/<key>` came back as `KeyDoesNotExist` so that if one node comes in and
creates the key, the other node would have their CAS fail because the value is
actually 0 now and 0 doesn't equal `usize::MAX`.

Once I addressed that bug, challenge 5b passed! 

For challenge 5c, I sat down and thought a little bit (and designed with Claude)
to think through what was needed to be 'more efficient'. 5b had about 14 msgs/op
to exchange with `lin-kv` before succeeding. One first attempt was to keep a
local cache to avoid making those read calls before sending a CAS. Because of
the safety properties of CAS, if the cached value was stale, I didn't need to
worry about losing correctness, I could just fall back to the original design
and issue a read if the cached value was stale. As I thought about it though, I
figured there may be a lot of places that ended up having to do this fallback
and that actually would **increase** the number of msgs/op because now the node
was sending an additional CAS message. Granted, it likely would amortize out and
actually give some improvements, but I was thinking there has to be an even
better way.

Claude suggested per-node key sharding. What this meant was each node would own
a portion of the key space and all operations would flow through that. In that
case, then the cache would be up-to-date and I could completely remove all of
those read calls before a CAS. However, I then just asked Claude, couldn't I
just have everything be done in memory for each of those nodes and stop using
`lin-kv` entirely? If only a single node was able to make operations to `lin-kv`
on behalf of a subset of the key space, then why not just store a local
key-value store (`HashMap`) per node and combine the 5a and 5b solutions? Claude
gave me the confidence that it should work and so I got to work. (Additional
note: since the workload didn't require partition tolerance, I could get away
with this. Adding partition tolerance would have required a lot more bookkeeping
(ACKs) and would likely have increased msgs/op).

For hashing, I used Rust's `DefaultHasher` which made it deterministic across
nodes. That is very important because using something like a time-based seed
would have caused each node to likely compute a separate hash value per key and
then there wouldn't be consistency across the cluster. `DefaultHasher` ensured
the same hash was computed for each key.

With this, the only main change from 5a was determining if the node could handle
the operation or have to ask another node for help. The approach I took was:
have a node complete as much of the operation as it could locally, batch up all
the keys that needed to go to other nodes, and then send that information off to
a node. Once that node received a peer request, it would be able to quickly
gather the results and send it back to the original node. Once the original node
(that the client reached out to) aggregated the results, then it responded to
the client request. 

It was important that I waited until all of the peers finished up their work
before responding to the client because that allows for the workload to be
sequential per key. I had to make sure that the values made 'sense' between a client
sending a request and receiving a response. The reason that is important is
because once a, say Poll, request came into a node, it would gather up its
current log values for the specified offset and then send out the request for
the keys that node didn't own. In that time waiting for the other nodes, the
values for that node could change (like another client adding a new value to one of
the keys the Poll was responding to). As long as that node doesn't respond to
the client who made the poll request, the node doesn't need to re-update its
response for keys it owns when the other nodes return their value to make it
remain sequential per key. The client still hasn't received a response from that Poll
request, so it is perfectly reasonable to set the total order of that operation
before the new Send came in. Since the poll started before on the wall clock
time, it is allowed to either send the fresh value that occurred throughout the
duration of its operation or the value before. The only requirement for a Poll
that starts at time t is that all operations that had **completed** for all s <
t showed up in the poll. These responses are not 'linearizable' because it is
still a stitched-together snapshot that allows concurrent writes within the gap.
The solution, with the single leader per key, makes sure the operations on the
keys that node owns is sequentially consistent. The multi-key operation is not
'atomic' but does provide those consistency guarantees per key since each node
owns a key.

Once that was done, I still had a correct implementation and drove msgs/op down
to 3.82. That made complete sense because it effectively meant every operation
required a little bit of communication (request other node for operation,
receive response for each node).

The final code for 5c is [here](https://github.com/carterburn/Gossip-Glomers/tree/main/challenges/kafka).
If you want to see the 5b solution, it is [here](https://github.com/carterburn/Gossip-Glomers/tree/93584aace4b32fdf9ad582abae3672c902850ad3/challenges/kafka)

## Challenge 6
The final challenge! For this challenge, a transactional key-value store needed
to be created. The key was that the two isolation levels required were Read
Uncommitted and Read Committed. I learned in this process that both of those
isolation levels are described in terms of what phenomena is *not* allowed. For
example, for read uncommitted, the system just can't allow dirty writes, meaning
if two transactions occur concurrently, the system has to be consistent with
some serial order of the two transactions.
What this means in reality is that there can't be a mixture of each transaction's writes that
remains. For example, if two transactions t1 and t2 have these sets of writes:
`t1: [x -> 1, y -> 2]` and `t2: [x -> 3, y -> 4]` the final state of the system
as a whole has to be either `[x=1, y=2]` or `[x=3, y=4]`. There can't be a weird
in-between like `[x=1, y=4]`. For 6c, the isolation level is bumped to Read
Committed. This prohibits: 

1. aborted reads: one transaction can't read the value written by an aborted
   transaction
2. intermediate reads: one transaction can't read an 'intermediate' value of
   another transaction (this means a transaction should **only** read final
values from other transactions)
3. circular information flow: two transactions can't have write-read or
   read-write dependencies (from fly.io: For instance, transaction T1 writes x = 1 and reads y = 2, and transaction T2 writes y = 2 and reads x = 1)

Additionally, in my research with Claude, I learned about HAT framing from Bailis et al. What this
said was that Read Committed is the highest isolation level achievable while
staying totally available under a partition. That meant
that you can't have a totally available key-value store (challenge 6) that provides
higher isolation than Read Committed. Anything above that requires some sort of 
coordination with other nodes. 

So, on to the challenges, 6a first was the single node version. This meant there
was no coordination that needed to happen. The important decision made there,
though, was how to apply transactions to the local state. I ended up going with
a scratch key-value storage to use for a transaction's scratch space that would
then get applied to the global store. What this meant is
that reads within a transaction would first consult this scratch buffer and then
the global store. Once the transaction was over, that scratch `HashMap` would then be applied
to the global store (in the single node) and then respond to the client. 

6a passed easily (as it is supposed to) so then it was time for 6b. With 6b, the key insight
was to start tracking transactions throughout the cluster with a unique-id per
write within a transaction. That would give the entire cluster a means to construct a total
order throughout the running of the cluster. This also meant there was a 'last-write-wins'
situation for the keys that were written during a transaction. The 'tag' for each write (as I called it) was: `Tag(per_node_seq, node_id)`.
I completely stole the unique-id solution (challenge 2) to use it here. The tags
would be lexicographically ordered, giving an ordering to any write that
occurred. Again, I used async gossip to push out new transactions to the other
nodes. That meant on every 'tick' in the runtime, the node would push out
transactions that occurred since the last tick. To ensure the node combatted
partitions, this also meant ACKs needed to float around to confirm each node
received the gossiped transactions. The node would simply store an outbox keyed
by each node along with unacked transactions (writes really only mattered) and
then would remove it on successful ACKs. 

Anytime a transaction came in (either from a client or from a peer gossiping),
there was a 'max-by-tag' rule that applied for each write to determine if it should get added
to the global store that the node was tracking. This is what ensured that there
was a 'total order' across the system and ensured that all of the nodes were
eventually getting to the right final state. The rule just viewed the current
`Tag` associated with a `(key, value)` pair and would only stomp over that key
in the global store if the new `Tag` was 'higher'.

6b ran very well and got the win. Then, I consulted Claude to start designing
the 6c solution and it led me to realize that my solution actually would likely
work for 6c. In the 6b solution, there was no abort path (I never provided the
option to abort a transaction), so aborted reads was already satisfied because
there was no such thing as an aborted transaction. The intermediate reads also
was satisfied because of the scratch buffer. Any intermediate values would be
overwritten in the course of the transaction and the final state was the
**only** thing that was gossiped to peers. Then, the circular information flow
was handled with the tagging of `(key, value)` pairs. What this meant was my 6b
solution was actually a 6c solution and that anything with Read Committed also
satisfies Read Uncommitted.

I ran the 6c workload and lo and behold, it passed. I finished up Gossip
Glomers!

You can find the 6b/c solution [here](https://github.com/carterburn/Gossip-Glomers/tree/main/challenges/txn).

## Review
Overall, I thought these challenges were awesome. They forced me to learn some
new material that I hadn't been exposed to in distributed systems courses I had
taken previously. Distributed systems is quite the vast subject area and a
single semester can't cover it all in depth. Gossip Glomers did a great job at
pinpointing difficult problems and solutions and tested them out. 

A few things I noticed that recurred:

1. Simpler designs tended to win: there were many times that I started with
   over-engineering a solution. I learned very well through this process that
its best to start with a simple solution that seems to 'work', measure the
result, and then make changes based on that result. It's a classic thing I've
heard time and time again, but it did come up here and I felt it. I assumed
"distributed systems is hard, it must be a tough solution", but in reality,
there were a lot of times that adding in a bunch of additional checks didn't
help the situation and wasn't required.
2. Start with single node: I appreciated that these challenges began with single
   node approaches. The single node approach allowed me to understand the
workload and the task at hand. What it did was highlight the decisions that I
had to make and specifically, *which things needed to be made across multiple
nodes*. Sometimes, there are things that don't require cross-node coordination
(like the scratch buffer in challenge 6), but if I hadn't started with the
single node version, I may have tried to replicate that information across. In
the end, I just needed to replicate the final state to make it known to the
cluster and it actually saved me from over-engineering and potentially saved me
having to write a separate 6c solution because I would have likely introduced
intermediate reads into the equation. In the future if I'm designing a
distributed system from scratch, I will absolutely begin with the single node
version and then gently ramp up the coordination to add more nodes when needed.

### Quick Note on Use of AI
I used Claude during this to be my 'instructor'. It's not easy to find someone
who would walk you through design decisions and point out issues when going
through something like this. I instructed Claude to never give me any code
snippets and rather just walk me through solving the problem myself with the
Socratic method. What this did was allowed me to go struggle with the concepts
first, attempt to code the solution myself, and then use Claude as a checker. 

I would say that it performed at about 85% for this task. There were a few times
that Claude couldn't help itself and just splatted out the code snippet I needed
help on. And there were also times that it tried to let me continue on without
understanding the theory fully. However, I would say that generally using Claude
this way allowed me to not only understand the distributed systems theory better
but also ensured that I was able to properly design the Rust crates in a way
that would be 'professional.' Claude allowed me to struggle on things like
functional iterator calls and designing structs and whatnot. That was part of
the key learning objective. I not only understood distributed systems better,
but I also became a better Rust programmer in the process. 

As a result, the code is all handwritten and Claude stuck around to simply
provide guidance and direction when I needed assistance.

That's it for Gossip Glomers! The things I'm currently still working on:

- FoundryDB: I promise more parts to the series will be coming out shortly. The
  issue: I really didn't know how much effort or theory goes into writing a
DBMS. I didn't feel like cstack's original series allowed it to really sink in
and other tutorials usually abstract away storage layers. I think cstack's posts
are great, but SQLite uses 'index-organized storage' meaning it uses a normal
index data structure and stores the values in the leaf nodes of that structure.
I didn't realize that that wasn't 'standard' and that slotted pages
('tuple-oriented storage') was the baseline. I wanted to get more of the
information so I have decided to pause and work on CMU's 15-445 course. I can't
share the code or really talk about anything in the blog from that course due to
CMU's academic policy (and not wanting to be on the [Wall of Shame](https://github.com/cmu-db/bustub/blob/master/WALL-OF-SHAME.md)),
but I can use it to upgrade my knowledge on DBMS engineering and that should
enable the FoundryDB post to be a better overall post. The main kicker is that I
don't know a whole lot of 'modern C++'. It shares similarities to Rust, but it seems to have
a good amount of gotchas. As a result, I'm upscaling my modern C++ knowledge and
then want to tackle at least the first two assignments in the course before
resuming FoundryDB. 
- AWS Serverless: I'm also trying to increase my abilities to make serverless
apps on AWS in Rust. I'm reading "Crafting Lambda Functions in Rust" on the side and want
to create a small serverless app in the near future. I'll write about that when
I get to it.

All that said: the focus is CMU's course right now and in parallel learning more
about really doing a lot on AWS serverless. I've used it before, but I know
there are things I can learn and improve on.

Until next time!
