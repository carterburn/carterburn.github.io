---
title: "New Series: Let's Build a Simple Database...in Rust"
layout: single
tags:
    - database
    - sqlite-series
---

Hey, all! Back at it again trying to write posts. While I do believe the Roundup
posts are great and will hopefully force me to stop coding and write about what
I'm doing, I wanted to come up with a solid series of developing something from
scratch. I've been intrigued by databases and their internals for awhile now and
one of the more fascinating posts that I have read about database internals is
from [cstack on Github](https://cstack.github.io/db_tutorial/). That tutorial
walks you through making a minimal sqlite clone in C. I have written a fair bit
of C in my time (refer back to the [about](/about) page on offensive security
tools, they are predominantly in C) and even started working through the code
but I eventually plateaued and stopped working through it. I thought a way to
motivate me to get through it and really learn was to write that clone in Rust
with all of the challenges of the borrow checker, ownership, and the like.

The plan is to try to follow the tutorial's fifteen parts as closely as
possible, but in Rust. I'm also going to attempt to avoid dependencies at all
costs for two reasons:

1. SQLite itself prides itself on minimal dependencies. I've heard on a podcast
with the creator that with no dependencies comes freedom to do what you want;
it's like backpacking in the wilderness, you're all on your own.
2. Using something like the `BTreeMap` in the Rust standard library feels like
cheating. We have to struggle through this!

## Extensions 
I plan to make a few extensions to the original at the end (if I'm up for it).
One of them would be to add the Write Ahead Log (WAL) functionality of SQLite.
This functionality is a key component of allowing SQLite to be distributed (many
distributed SQLite extensions and components leverage the WAL). With that, I may
be able to tie together my Raft implementation with this small sqlite clone. 

I'm going to try to do my best and make it close to SQLite. I know I won't get
there but the tutorial seems to focus in on the key components of SQLite and I
may want to add some of the other features.

Other extensions are in my mind as well (maybe some python bindings to allow the
use of this clone within python) and will be explored as we make our way
through.

## Table of Contents
Here is the table of contents for the entire series. I wanted this to be higher
in the post. See below this section for Getting Started! 

These are going to take the form of the original series by cstack mostly. More
will be added as we work our way through. 

1. Part 1 - SQLite Introduction and Setting up the REPL in fdb --> Coming Soon!
2. TBD

## Getting Started
To get started, I'm going to create a new Rust crate. I'll start by creating
this as a library with a binary for a command line tool (like the `sqlite3`
binary you can use to view SQLite databases from the command line). The purpose
for this is extensibility with potential future extensions. It makes it a lot
easier to start with a library and port that than try to undo a binary.

We're calling this FoundryDB (foundry -> Rust get it?):

```bash
cargo new foundrydb --lib
# get the simple binary file created
touch src/main.rs
```

Then, update Cargo.toml to look like this:
```toml
[package]
name = "foundrydb"
version = "0.1.0"
edition = "2024"

[lib]
name = "foundrydb"
path = "src/lib.rs"

[[bin]]
name = "fdb"
path = "src/main.rs"

[dependencies]
```

I'll leave the default `lib.rs` code of `add` and add some scaffolding to
`src/main.rs` to make sure they're playing nice.

In `src/main.rs`:
```rust
use foundrydb::add;

fn main() {
    println!("Inside fdb! {}", add(1, 2));
}
```

Then, we can verify we can `test` and `run` the library and binary respectively:

```bash
cargo test
cargo run
```

Of course, we now push to Github. You can find the repo [here](https://github.com/carterburn/foundrydb).
We'll add a README later on.

In the spirit of cstack's original post, I'll include the SQLite Architecture
diagram as referenced at https://www.sqlite.org/arch.html.

![sqlite architecture](https://cstack.github.io/db_tutorial/assets/images/arch2.gif)
