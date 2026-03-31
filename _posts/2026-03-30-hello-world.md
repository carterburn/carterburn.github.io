---
title: "Hello World"
layout: single
tags:
    - intro
---

Hello world from Github Pages! 

As seen in the [about](/about.md) page, I set this blog up to start
getting some written content out to the world. The greatest barrier to entry for
me has been trying to capture a "perfect" post and spending so much time getting
a post there that I end up getting distracted and never actually publish it. I
intend to change that with this refreshed site. I'm planning to write about
things I'm currently working on, design decisions, problems, etc. and then just
post it. Minimal editing, minimal review, just brain dump a post and publish it. 

I fully recognize the importance of technical writing and blog posts. It would
take me ages to list the countless blog posts that have guided me through
solutions and I hope that my brain dumps can be useful to someone down the line.
If they're not, then at least this is an attempt to force myself to write
technical content and get it on the Internet.

## Current State
Another one of my pitfalls I've learned through personal projects is my ability
to finish them. I am not going to make any promises that I will do that (or that
this blog will help me finish them). 
> If we're being honest, is a project ever "done"?! 
Currently, I'm working on implementing the Raft consensus algorithm in Rust. I'm
taking a sans-i/o approach to it and focusing on the algorithm correctness at
the moment. I plan to finish that soon and then write the event loop (where the
I/O will actually start to happen) along with a minimal key-value server on top
of Raft (to give a replicated, fault-tolerant KV store). 

I've also been messing around with Code Crafters, specifically the Redis
implementation. I actually got up to the point where multiple servers would
interact and that made me switch over to writing the Raft library. I'm not sure
if I'll combine the two (mostly because I like the idea of a raw KV store
instead of adding the Redis-specific items), but maybe that'll happen. Check out
my progress so far (here)[https://github.com/carterburn/codecrafters-redis-rust].

I've also been working on a tunneling tool (there's those security tools again)
that allows a user to tunnel into a network using a tun device on their host
machine over a QUIC connection. That isn't public yet because it only supports
TCP tunneling at this point and it doesn't have tests nor does it provide
security with the self-signed certs. I'm uncertain if I'll continue that at the
moment, but it is still a fun project I may share in the future.

After I get through Raft, my plan is to try my hand at the Gossip Gloomers
Distributed Systems problems found on [fly.io](https://fly.io/dist-sys/). I had
Claude generate some reading materials to help me through because while I did
take a distributed systems course for my graduate degree, it appears there were
some gaps in that course. I'll definitely blog about those challenges here.

## Previous Work
Two main things to point out as previous Rust work:

1) I wrote a simple crate for a socks5 proxy in Rust. It's pretty bad, but it
was honestly my first attempt at writing any sort of networking code in Rust
using tokio. You can find it on (crates.io)[https://crates.io/crates/socksprox]
and the source on [Github](https://github.com/carterburn/socksprox). I have to
say, greater than 3k downloads is still pretty cool, even if that's a bunch of
crates.io mirrors downloading the crate every once in awhile. 
2) I wrote a userland exec binary in Rust a little bit ago. I called it santa
because it only loads ELF binaries. I drew inspriation from a Python
(implementation)[https://github.com/anvilsecure/ulexecve/tree/main]
of userland exec and a Rust (one)[https://github.com/io12/userland-execve-rust/tree/main]. 
This is another one of those security tools that allows someone to load up a 
binary inside an existing process's memory (santa itself) without having
the kernel do it. It can load binaries from files, stdin, or from a remote URL.
While writing this blog, I forgot I had to add a few things to the README, so I
sent Claude off to go do that. Check out the repo
(here)[https://github.com/carterburn/santa].

## Quick Word on AI (blog posts in 2026 must include)
In a lot of my projects, I do leverage Claude Code. My approach to using AI,
though, is to use it like a teacher. For example, in my Raft implementation, I
don't have Claude write any code, I just have it give me "tasks" as if it were
an assignment and then review my work and give me guidance on places I can
improve. I think this has worked well for me because it allows me to ensure I
understand what I'm writing and forces me to write idiomatic code. I think it's
making me better without relying too much on Claude to write the code for me. I
definitely recognize the time and place to have Claude do that work and have
considered a few projects to let Claude go do its thing, but I do fear if I do
that too frequently, I miss out on the opportunity to truly learn and then won't
be able to effectively use AI coding assistants. The power of an AI coding
assistant is when the human can effectively guide (and check) the assistant,
which requires deep understanding of the topic at hand. I need to get to that
point first, so I use Claude as my personal teacher (because I don't really have
one). 

Hopefully this is a blog that will interest you! We'll see how it goes. Until
next time!
