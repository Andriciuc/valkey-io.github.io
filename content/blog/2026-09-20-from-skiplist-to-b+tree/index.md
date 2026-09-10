+++
title = "From Skiplists to B+ Trees: Making Valkey `ZSETs` More Memory Efficient"
date = 2026-09-20
description = "How Valkey 9.2 replaces the skiplist used by large `ZSETs` with a cache-friendly B+ tree, reducing memory overhead while improving several ordered-set operations." 
authors =  ["dragosandriciuc","rainvalentine"]
[taxonomies]
blog_type = ["Technical Deep Dive"]
+++

A `ZSET` can spend more memory describing how its elements are ordered than storing the elements themselves.

In Valkey 9.2, however, that changes: the skiplist is replaced by a high-fanout B+ tree called **fbtree**.

fbtree combines the high fanout and contiguous storage of a conventional B+ tree with several optimizations designed for Valkey's workload and memory allocator, including feature-based routing, SIMD searches, linked leaves, and fast paths for sequential inserts and pops.

Let's take a look on why this change happened, how it works, and why it's important for memory optimization moving forward. A benchmarking example is also provided, to get a clearer view of the amount saved if you would apply this change.

## Why replace the skiplist?

Large `ZSETs` (anything past the listpack threshold) are backed by a skiplist paired with a hashtable. The hashtable gives O(1) lookups by member, the skiplist gives O(log n) ordered lookups by rank or score. That pairing has worked well for years, but the skiplist carries a hidden cost that is often overlooked. It stores exactly one element per node, scattered across memory as separate allocations connected by pointers.

That's not a Valkey-specific problem, a 2025 systems paper studying in-memory indexes for key-value stores found that traditional skiplists incur roughly 2.4–4.8x more cache misses than a comparable B-tree on equivalent workloads, translating to 2–8x lower throughput, that is the cost of storing one element per node instead of several packed together[^1].

[^1]: [Bridging Cache-Friendliness and Concurrency: A Locality-Optimized In-Memory B-Skiplist](https://arxiv.org/abs/2507.21492)

The mechanical reason is straightforward: a modern CPU can fetch several adjacent cache lines nearly as cheaply as one, but only if the data is actually adjacent. A B+ tree node that spans a few cache lines costs barely more to read than a single cache line would. A skiplist node has no such luck as each one is its own allocation, scattered wherever the heap happened to put it, so every hop is a fresh trip to memory.

Valkey's skiplist pays for this in raw memory too. Every node carries forward and back pointers (16B), plus a variable number of additional level pointers whose count is determined probabilistically, with a 25% probability of adding another level, which works out to an expected 1.33 extra levels per node. Each level costs 8B for the pointer plus 8B for a span value, landing around 37B of pure structural overhead per item, before a single byte of score or element data [^2].

[^2]: ([Issue #3166](https://github.com/valkey-io/valkey/issues/3166)).

There's a cleaner way to think about that 25%-per-level rule: it gives the skiplist an effective branching factor of about 4. A B+ tree with a fanout of 61 (each node can hold up to 61 entries or child pointers) searches a much larger portion of the dataset at each level. Both structures retain O(log n) search complexity, but the B+ tree has substantially fewer levels to traverse and therefore fewer opportunities for expensive memory fetches.

This also isn't the first time `ZSET` memory has gotten smaller. An earlier release folded Valkey's `dict` into the newer `hashtable` implementation, and the b+tree change stacks directly on top of those savings rather than replacing them [^3].

[^3]: [A new hash table: Technical Deep Dive](https://valkey.io/blog/new-hash-table/)

## What replaced it: fbtree

Short for FB+ Tree, or Feature B+ Tree, has inner nodes that store a small "feature" for each child: four bytes taken from the child's anchor value. Because the anchors often share a common prefix, fbtree stores that shared prefix separately and uses the four feature bytes to distinguish the children. The implementation can then compare those features in parallel using SIMD, often identifying the correct child before fetching the child node itself. If the features uniquely identify a child, the search can descend immediately; otherwise, they still narrow the range that needs a full binary search.

Structurally, the tree uses a 61-way fanout, with leaf and inner nodes sized to fit jemalloc allocation classes without wasting space. Leaf nodes (which hold the actual scored elements) are linked together in a doubly-linked list, so range operations like `ZRANGE` can walk forward without climbing back up the tree at every step. A couple of targeted shortcuts round it out, such as a fast path for pushing and popping at either end of the set (benefiting commands like `ZPOPMIN`) and an efficient way to delete a contiguous range of elements without rebuilding the surrounding structure.

The payoff shows up in how the CPU reads it. A 512-byte leaf spans eight typical 64-byte cache lines, but those lines are adjacent. Hardware prefetching can therefore bring much of the node into cache as the CPU scans it. The skiplist has the opposite access pattern: each node is a separate allocation, so following the structure means chasing pointers to unrelated memory locations. Same O(log n) complexity, much smaller constant factor.

Another change happens at the leaf level. Instead of storing the score and member separately, fbtree stores them together as a single packed value: the normalized 8-byte score followed by the member bytes. This keeps the data needed for comparisons together and removes another level of pointer indirection.

## What actually changed for you

**Functionally, nothing changes.**

The change is internal: `ZSET` commands and their behavior remain unchanged. The one visible difference is what `OBJECT ENCODING` reports for a large sorted set return `btree` instead of `skiplist`. If you have monitoring, tests, or tooling that checks for the literal string `skiplist`, that's one place to update. Small `ZSETs` under the listpack threshold are unaffected either way, since they never used the skiplist encoding to begin with.

## Presenting the numbers

The benchmarks below are from the [merged implementation](https://github.com/valkey-io/valkey/pull/4206). They were run on a Graviton3 c7g.metal system with 64 cores, nine I/O threads, a pipeline depth of 10, and a 3-million-member `ZSET`. Each test was repeated five times; the reported confidence intervals were ≤2% for all commands except `ZRANDMEMBER`.

**Note: `ZSCORE` and `ZRANDMEMBER` barely move because they use the companion hashtable rather than the ordered index.**

| Command | Throughput improvement |
|---|---|
| `ZADD` | +105% |
| `ZREM` | +76% |
| `ZCOUNT` | +27% |
| `ZRANK` | +19% |
| `ZRANGE` | +6% |
| `ZRANDMEMBER` | +2% |
| `ZSCORE` | +2% |
| `ZRANGEBYSCORE` | +1% |
| `ZPOPMIN` | ~0% |

The pattern makes sense once you look at what each command actually does: `ZADD` and `ZREM` reposition elements in the tree, so they benefit most directly from fbtree's shallower structure and fewer pointer updates. Range operations see smaller gains because once the index traversal becomes cheap, producing and returning the requested elements becomes a larger part of the total cost.

Memory tells a similar story, and it depends on how the data got there. Inserting 5 million 20-byte members sequentially dropped average per-item memory from 50.3B to 28.5B, a **43% reduction**. The same 5 million members inserted in random order dropped from 50.3B to 32.0B, a 36% reduction, slightly smaller because random insertion leaves more partially-filled nodes than a sequential fill does.

<!--Note: Diagrams will be added soon after this branch is published.-->

## How the migration was validated

Replacing a core data structure in a mature database is less about implementing the new structure than proving that it behaves exactly like the old one.

An OrderedIndex interface was first introduced between `ZSET` operations and the underlying data structure. The existing behavior was then captured in a shared test suite, allowing the skiplist and fbtree implementations to be tested against the same contract.

[PR #3840](https://github.com/valkey-io/valkey/pull/3840) introduced this abstraction and migrated the `ZSET` call sites before the fbtree implementation replaced the skiplist.

The final implementation was then validated with unit tests, integration tests, property-based and fuzz testing, and full-server benchmarks, 302 new unit tests plus 21 new integration tests covering the new encoding [^4].

[^4]: [PR #4206](https://github.com/valkey-io/valkey/pull/4206)

## Known limitations

One gap worth knowing about if you run delete-heavy `ZSET` workloads: fbtree doesn't yet merge or rebalance nodes on delete. If your workload adds and removes elements at similar rates over a long period, leaf nodes can end up sparse, which is technically correct, but no longer packed as tightly as a fresh insert would be.

Background compaction is planned as a follow-up. If you're running a workload with heavy churn, it's worth watching `MEMORY USAGE` over time rather than assuming the benchmarks above hold indefinitely.

## Try it in 9.2

The important takeaway is simple: **the commands didn't change**, but the cost of keeping them fast did.

The goal wasn't simply to make the data structure faster in isolation: the implementation was repeatedly benchmarked against the existing skiplist through the full Valkey server, with optimizations added until fbtree matched or exceeded the skiplist across the tested ZSET commands.

If large `ZSETs` are a significant part of your Valkey workload, test Valkey 9.2 against your real workload, particularly if you're dominated by `ZADD`, `ZREM`, `ZRANK`, or `ZCOUNT`.
