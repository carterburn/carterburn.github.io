---
title: "Gossip Glomers - Challenge 1 - 3"
layout: single
tags:
    - gossip_glomers
    - rust
    - distributed systems
---

Hello, all! Following the completion of my Raft implementation, I wanted to keep
plugging away at distributed systems projects and challenges. 

> Don't forget about the SQLite clone in Rust series. That's still running but I
> like to switch back and forth from time to time! The [intro](/foundrydb-intro)
> kicks off the series!

The number one place I kept stumbling across was fly.io's Gossip Glomers
challenges found [here](https://fly.io/dist-sys). It is a set of challenges
meant to stress test your ability to solve interesting problems in the
distributed systems world. Of course, I wanted to do these challenges in Rust.
The challenges leverage 
[Maelstrom](https://github.com/jepsen-io/maelstrom) which is a "workbench" for
learning distributed systems. It uses
[Jepsen](https://github.com/jepsen-io/jepsen) under the hood to test toy
implementations. 

> One note: there are a few more 'workloads' or challenges that exist within the
> Maelstrom repo that I'll likely give a shot after Gossip Glomers. One of them
> looks like a linearizable key-value store which we nearly have with our Raft
> implementation! So, I definitely plan to do that!

Maelstrom is pretty awesome and I think it is a great way to prototype some of
these algorithms with a little less of the headache of network sockets, setting
up infrastructure for a test, etc. It just spins up a few processes and
communicates over stdin/stdout. Maelstrom will send out some JSON objects as
"messages" over STDIN and the node implementation will then write any messages
it wants to send to STDOUT. This definitely lends itself to a sans-I/O approach
where there is a 'handler' that manages reading messages from stdin and writing
messages to stdout and then drives a state machine with those messages. The
state machine will just take in certain parsed messages and then output
additional messages. (Some of the challenges also require some sort of timeout
and that will definitely add some complexity you'll see below).

At the time of writing, I'm working on [Challenge 3d - Efficient Broadcast, Part I](https://fly.io/dist-sys/3d/).
I already finished the preceding challenges and will give a rough overview of
the solutions to those as well as discuss the shared runtime (event loop) I wrote 
to make writing challenges a bit easier. Let's start with Challenge 1: [Echo](https://fly.io/dist-sys/1)!

# Challenge 1 - Echo
The point of this challenge is mostly to understand the Maelstrom protocol. It's
a way to get used to how to interact with the test harness. The 'challenge' is
to receive "echo" messages from clients and just echo the payload the client
sent back to them. Fairly simple and a bit of the "hello world" for Maelstrom
and distributed systems. 

The main point with the challenge is understanding the
[protocol](https://github.com/jepsen-io/maelstrom/blob/main/doc/protocol.md).
The protocol itself is fairly simple, every communication has an "envelope" if
you will that looks like this:

```json
{
    "src": "<src-node>",
    "dest": "<dest-node>",
    "body": <message-specific-object>
}
```

Every challenge will define a set of RPCs that make up the body key in the
envelope JSON object. Another key point in the protocol that the Echo challenge
helps resolve is the "init" message. At startup, Maelstrom sends a single init
message at the start which provides each node its node ID and a list of the
other nodes. The init message sent by Maelstrom will have a body that looks like
this:

```json
{
  "type":     "init",
  "msg_id":   1,
  "node_id":  "n3",
  "node_ids": ["n1", "n2", "n3"]
}
```

And the node needs to simply respond with a "init_ok" message back as a reply to
that message (using the "in_reply_to" key that should indicate replying to a
message with that "msg_id"). That body looks like this:

```json
{
  "type":        "init_ok",
  "in_reply_to": 1
}
```

For the echo workload itself, the body that clients will send looks like this:

```json
{
    "type": "echo",
    "msg_id": 1,
    "echo": "Please echo 35"
}
```

And the response should be like:

```json
{
    "type": "echo_ok",
    "msg_id": 1,
    "in_reply_to": 1,
    "echo": "Please echo 35"
}
```

Notice how the response is effectively a mirror of the original body, it just
changes the "type" and sets the "in_reply_to" to the "msg_id" of the request. 

At this point, I could quickly wire up a program that is able to perform this
challenge. It was tempting to write a library and get things ready for all of
the future challenges, but I opted to just start with a simple solution to
challenge 1 and make a shared library after challenge 2 reveals the 'repeated'
patterns. Reading from stdin and stdout is fairly straightforward and obviously
the actual challenge is difficult, but I'll introduce how to use `serde` to
construct these message types. 

For the envelope, a simple struct works to define the overall JSON object:

```rust
#[derive(Debug, PartialEq, Serialize, Deserialize)]
struct Message {
    src: String,
    dest: String,
    body: Body,
}
```

Deriving `Serialize` and `Deserialize` allow the use of `serde_json` later on
for actually creating the JSON object. For the `Body` of this challenge, it's
defined as:

```rust
#[derive(Debug, PartialEq, Serialize, Deserialize)]
#[serde(tag = "type", rename_all = "snake_case")]
enum Body {
    Init {
        msg_id: usize,
        node_id: String,
        node_ids: Vec<String>,
    },
    InitOk {
        in_reply_to: usize,
    },
    Echo {
        msg_id: usize,
        echo: String,
    },
    EchoOk {
        msg_id: usize,
        in_reply_to: usize,
        echo: String,
    },
}
```

The key there is the `#[serde(tag = "type", rename_all = "snake_case")]`. What
that does is first makes the `enum` "internally tagged", meaning it collapses
the variants and the enum name into a single object and places the `enum`
variant name as a key in that object with the tag "type". Then, all of the
variants are renamed with snake_case as opposed to the PascalCase standard in
Rust enum definitions. The 
[serde docs](https://serde.rs/enum-representations.html#internally-tagged)
have more information, but essentially this will make an "Echo" variant look
exactly as its defined in the Maelstrom protocol.

Once that was done, a quick switch-a-roo on the input allows this test to pass.
You can find the code for the challenge [here](https://github.com/carterburn/Gossip-Glomers/tree/main/challenges/echo).

# Challenge 2 - Unique IDs
This [challenge](https://fly.io/dist-sys/2) is another 'simple' problem. The point is to have each node
capable of generating an ID for each client that is unique throughout the
cluster. The key insight here is that it doesn't need to be over-engineered with
consensus or anything like that; there is a simple solution. 

Every node gets a node ID that is unique among the cluster (no other node has
that ID) and then if each node ensures that it doesn't repeat a sequence number
from itself (with, say, a monotonically increasing integer that only changes on
each user request), then that ID is unique among the cluster. The format for
this ID will simply be "{node_id}-{seq_num}". 

What makes this challenge interesting is the shared runtime hoisted out. I'll
just talk about the key pieces and provide a link to the crate for reference.
The main idea is the `Node` trait which each challenge will define. I started
with a simple sync version but will just talk about the async version that
evolved for challenge 3 because it was required to add in a way for a node to
get notified on some tick interval to take actions in between Messages being
received from Maelstrom or other nodes. 

A `Node` is defined as:

```rust
pub trait Node {
    type Body: Debug + Serialize + DeserializeOwned;

    fn new(node_id: String, node_ids: Vec<String>) -> Self;

    fn handle(&mut self, src: String, body: Self::Body) -> Vec<Outgoing<Self::Body>>;

    // Optional methods for tick intervals
    fn tick(&mut self) -> Vec<Outgoing<Self::Body>> {
        vec![]
    }

    fn tick_interval() -> std::time::Duration {
        std::time::Duration::from_secs(3600)
    }
}
```

A `Node` must define a `Body`, which should just be an `enum` for that
challenge's RPC workloads (the message types the challenge cares about). Then,
there needs to be a `new` method that the runtime will call to share information
about the node's ID and its peers. Then a required method is `handle` which will
be called anytime a `Message` is received on stdin. There are also methods
related to ticks that have defaults for `Node`'s that don't want to participate
in ticks. The interval is set to an hour but will tick right away. I decided to
let it go and not force that first tick because there shouldn't be any logic
executed in a tick that is default. (This may be an oversight, but until I see
it as an issue, I'll leave it).

Then, I won't go into the event loop too deep because the code isn't particularly
interesting. The loop reads the init message from Maelstrom, calls `new()` on
the `Node` and then will select on either the tick interval (with
`tokio::time::interval`) or reading input from stdin. The `Node` will respond
with a `Vec` of `Outgoing` which is a simple struct that holds the destination
node ID and the `Body` defined by the `Node`. 

The runtime takes care of constructing the envelope or `Message` and is generic
over the `Node`'s `Body` associated type. `Message` looks like this:

```rust
#[derive(Debug, PartialEq, Serialize, Deserialize)]
struct Message<B> {
    src: String,
    dest: String,
    body: B,
}
```

Fairly simple, just generic over `B`. The actual constraints on `B` being
`Serialize` and `Deserialize` are on the trait definition itself. The runtime's
event loop then just provides the `Node` deserialized `Node::Body`'s and then
receives `Node::Body`'s back (in the `Outgoing` struct) and will wrap them up in
a `Message` and send through stdout. 

You can find the runtime crate [here](https://github.com/carterburn/Gossip-Glomers/tree/main/runtime-maelstrom). It may change up but I can't
imagine some of these main design decisions would change substantially. (The
link is the final async version of the runtime even though a sync version was
used briefly to test the concept with the echo challenge.)

I then wrote the unique IDs challenge using this crate and you can see that
[here](https://github.com/carterburn/Gossip-Glomers/tree/main/challenges/unique-ids). Again, it's fairly simple. Monotonic counter and then concatenate
the node's ID to that counter for a globally unique ID. 

# Challenge 3a - 3c - Broadcast
[Challenge 3](https://fly.io/dist-sys/3a) begins to put the test on distributed systems. In this challenge,
the goal is to allow clients to request a value be broadcast to the cluster and
the cluster must eventually learn that value (all nodes must have that value).
There are 2 primary RPC workloads from clients: "broadcast" and "read".
"Broadcast" is a client request to put a value in the cluster and "read" is to
get the entire set of values broadcasted thus far. Challenge 3a is a 'single
node' broadcast, meaning, there is only one node. The point is to just figure
out the workloads and then ensure the concept is handled. It was fairly easy and
just involved keeping a `HashSet` of the values. Challenge 3b stepped it up to
multiple nodes but in a "perfect" environment, meaning there would be no
partitions or errors in the system. The solution to this one is also fairly
simple where all that is needed is the ability to take that `HashSet` from the
initial challenge and just send out messages to each node to let them know they
received the message. 

To do this, I created a new variant in the `Body` called "Propagate" that looked
like this in its variant:

```rust
Propagate { messages: Vec<usize> } 
```

Anytime a "broadcast" was sent to a node, that node would send out a "Propagate"
message to all other nodes to ensure they added the value to their own set. Also
a fairly simple approach (but of course it is; there is a perfect network!).

The difficult ramps up a bit on [Challenge 3c](https://fly.io/dist-sys/3c). Here,
nodes can be partitioned off from the rest of the network meaning they might not
receive those "Propagate" messages. In that case, a node never has the opportunity
to learn that value and would never return the correct information on a "read" RPC from a client.

The solution for this is to add "ACKs" (or "PropagateOk") to check off when a node
has seen a particular value. The node will keep track of a list of values, per peer, that
the node has not ACKed. Then, on a `tick` interval (100ms was the choice here), the node will
attempt to re-send any values that are not ACKed to the specified peer. The node 
will also need to keep track of messages that it has sent and what values are
attached to those messages to properly clear the 'unacked' values. Here are the two 
structures used to track that:

```rust
unacked: HashMap<String, HashSet<usize>>,

sent: HashMap<String, SentRecord>,
```

`SentRecord` is just a struct that keeps the `msg_id` of a message and the
values that were sent in that message. I'll construct a "Propagate" message like this:

```rust
for (peer, unacked) in &self.unacked {
    if unacked.is_empty() {
        continue;
    }
    // we need to create a Message to send to this peer and store it in sent
    self.msg_id += 1;
    let values: Vec<usize> = unacked.iter().copied().collect();
    messages.push(Outgoing {
        dest: peer.clone(),
        body: Body::Propagate {
            msg_id: self.msg_id,
            messages: values.clone(),
        },
    });
    self.sent
        .entry(peer.clone())
        .and_modify(|record| {
            record.msg_id = self.msg_id;
            record.messages = values.clone();
        })
        .or_insert(SentRecord {
            msg_id: self.msg_id,
            messages: values,
        });
}
```

In a loop over the unacked values, the node will create a Message with the
values that aren't acked and place those in the outbox of the Node and then also
track that message. Once a node receives a "PropagateOk" (an ACK to its
"Propagate"), it can check those values off from unacked:

```rust
if let Some(record) = self.sent.get(&src) {
    if record.msg_id == in_reply_to {
        let Some(peer_unacked) = self.unacked.get_mut(&src) else {
            return vec![];
        };
        for v in &record.messages {
            peer_unacked.remove(v);
        }
        self.sent.remove(&src);
    }
}
```

If the response is a message the Node was tracking, it will remove the tracked
values from the unacked set and won't try to send those values again. This means
the node only accepts the last sent values and doesn't care about earlier
ACK responses. 

Here are the solutions for each problem (to the specific commits):

- [Solution 3a](https://github.com/carterburn/Gossip-Glomers/commit/105c2cfbabe4e712b76142e9929038693cbdca2a)
- [Solution 3b](https://github.com/carterburn/Gossip-Glomers/commit/1be60390a4a2a95fe55234e11440a9e61123aa20)
- [Solution 3c](https://github.com/carterburn/Gossip-Glomers/commit/7ce283710b598cf6af3df44be6455f8b9738ad9e)

# Challenge 3d
This is the [challenge](https://fly.io/dist-sys/3d) I am currently starting. I'll give a more detailed
overview of that in the next post. The key idea here, though, is that now we
have latency constraints and need to be smarter with how we send those Propagate
commands. This is where normal gossip protocols are going to come into play. A
node shouldn't waste time sending to all nodes and waiting on ACKs, etc. It
should start to spread a rumor of a new value to a subset of nodes who will then
spread to more nodes to propagate a value throughout the system quickly. I'll
give it more justice in the next post, but wanted to provide a quick overview of
the "getting started" posts to show you where I'm at. The next post I'll focus
more on the distributed systems problem itself (the gossip protocol) and less on
the infrastructure to solve the challenges.

Here goes nothing on 3d! See you in the next post to report what I find!
