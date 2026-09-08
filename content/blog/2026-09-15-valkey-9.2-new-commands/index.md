+++
title = "What’s New in Valkey 9.2 for Application Developers"
date = 2026-09-15
description = "Exploring a couple of Valkey's 9.2 commands as well as options developers can use to improve their workflow, older commands given new capabilities, and some real life scenarios where they would be useful in your deployment." 
authors =  ["dragosandriciuc"]
[taxonomies]
blog_type = ["Technical Deep Dive"]
+++

The arrival of Valkey 9.2 adds several changes that make application logic simpler. Instead of handling some conditions in your application code, making extra round trips, or maintaining your own bookkeeping, you can now ask Valkey to perform more of that work directly.

Let's look at a few of these changes through real application scenarios, from optimistic locking and stream cleanup, to writes that check their own preconditions, to replies that finally tell you what actually happened.

## `EXEC IFEQ | IFNE | NX | XX` for optimistic-locking workflows

Let's assume your application tracks account balances, and a transfer needs to update two keys together, but only if the account hasn't been touched by another transfer since your application last read it. This is a classic optimistic-locking pattern. The database reads a version number, then writes only if it hasn't changed.

Before 9.2, doing this safely inside a Valkey transaction meant using `WATCH` to watch the version key before you even opened `MULTI`, which costs pipelined clients an extra round trip before they've queued a single command.

Valkey 9.2 lets you make `EXEC` itself conditional. You can now attach transaction preconditions directly to `EXEC`, avoiding the extra `WATCH` round trip that trips up pipelined clients.

Consider the following example:

```bash
127.0.0.1:6599> WATCH ver{foo}
127.0.0.1:6599> GET ver{foo}
127.0.0.1:6599> MULTI
127.0.0.1:6599> SET ver{foo} 2
127.0.0.1:6599> SET mykey{foo}1 111
127.0.0.1:6599> SET mykey{foo}2 222
127.0.0.1:6599> EXEC
```

Now with the updated command, you can do:

```bash
127.0.0.1:6599> GET ver{foo}
"1"
127.0.0.1:6599> MULTI
OK
127.0.0.1:6599> SET ver{foo} 2
QUEUED
127.0.0.1:6599> SET mykey{foo}1 111
QUEUED
127.0.0.1:6599> SET mykey{foo}2 222
QUEUED
127.0.0.1:6599> EXEC IFEQ ver{foo} 1
1) OK
2) OK
3) OK
```

The real value shows up when another client changes `ver{foo}` after you've read it but **before** your `EXEC` runs. With the old `WATCH`-based approach, `WATCH` handles that detection with an extra round trip. Here, the same protection comes from the precondition attached at `EXEC` time:

```bash
# Client A:
127.0.0.1:6599> GET ver{foo}
"1"
127.0.0.1:6599> MULTI
OK
127.0.0.1:6599> SET ver{foo} 2
QUEUED
127.0.0.1:6599> SET mykey{foo}1 111
QUEUED
127.0.0.1:6599> SET mykey{foo}2 222
QUEUED
# Client B:
127.0.0.1:6599> SET ver{foo} 99
OK   (a concurrent write sneaks in)
# Client A:
127.0.0.1:6599> EXEC IFEQ ver{foo} 1
(nil)
```

Because `ver{foo}` no longer equals `1` by the time `EXEC` runs, the entire transaction is discarded. That leaves `ver{foo}` at Client B's `99`, and neither `mykey{foo}1` nor `mykey{foo}2` is ever written. For this specific optimistic-locking pattern, that's the same protection `WATCH` provides, without the extra round trip to set it up.

For more information, see the [EXEC command documentation](https://valkey.io/commands/exec/).

## `XACKDEL` and `XDELEX` for safe cleanup in fan-out stream consumers

Sometimes your application has more than one consumer group independently processing the same stream, one group logging events, another triggering notifications, and so on.

Before 9.2, cleaning up old entries meant either trimming the stream on a schedule and hoping every group had caught up, or writing your own bookkeeping to check every group's pending-entries list before deleting anything. Delete too early, and a slower consumer group loses messages it hasn't processed yet, that data is gone forever.

Valkey 9.2 adds two commands that build that check into the delete itself. `XACKDEL`, which acknowledges a message for one specific group **and** deletes it in the same call, and `XDELEX`, which deletes stream entries directly. Let's look at how both of these work.

### `XACKDEL` to delete acknowledged messages

`XACKDEL` is a stream command that acknowledges one or more messages and (conditionally) deletes them from the stream.

This is particularly useful when multiple consumer groups independently process the same stream and you need to reclaim entries without deleting messages that another group still needs.

With `ACKED` mode, the message is deleted only once no consumer group still needs it, meaning none has it pending (each has either acknowledged it or never picked it up), and no group can still deliver it later.

The command supports the following deletion modes:

- `KEEPREF` (default, implicit): acknowledges and deletes messages immediately, leaving `PEL` references in other groups
- `DELREF`: acknowledges, deletes, and forcibly removes PEL entries from all other groups
- `ACKED`: deletes a message only once no consumer group still needs it, meaning none has it pending (each has either acknowledged it or never picked it up), and no group can still deliver it later

`XACKDEL` combines acknowledgment and conditional deletion into a single command, whereas `XDELEX` below handles the deletion separately. See `XDELEX`'s example below for `ACKED` mode in action.

### `XDELEX` to delete stream messages

`XDELEX` is an extension of the Valkey Streams [`XDEL` command](https://valkey.io/commands/xdel/) that allows you to delete one or more stream messages with more control over how those message entries are deleted concerning consumer groups.

The command supports three deletion modes:

- `KEEPREF` (default): deletes the stream entry but leaves PEL references intact in all consumer groups
- `DELREF`: deletes the stream entry and forcibly removes it from all consumer group PELs
- `ACKED`: deletes a message only once no consumer group still needs it, meaning none has it pending (each has either acknowledged it or never picked it up), and no group can still deliver it later

**Note:** The command returns a per-ID integer array: `1` for deleted, `2` for exists-but-not-yet-deletable (`ACKED` mode only), and `-1` when the message wasn't found, wasn't delivered to the group, or was already acknowledged by it.

Consider the following example of `ACKED` mode in action:

```bash
127.0.0.1:6899> XADD s4 * a 1
"1788346731590-0"
127.0.0.1:6899> XGROUP CREATE s4 grp 0
OK
127.0.0.1:6899> XREADGROUP GROUP grp cons1 COUNT 10 STREAMS s4 >
1) 1) "s4"
    2) 1) 1) "1788346731590-0"
            2) 1) "a"
               2) "1"
127.0.0.1:6899> XDELEX s4 ACKED IDS 1 1788346731590-0
1) (integer) 2
127.0.0.1:6899> XLEN s4
(integer) 1
127.0.0.1:6899> XACK s4 grp 1788346731590-0
(integer) 1
127.0.0.1:6899> XDELEX s4 ACKED IDS 1 1788346731590-0
1) (integer) 1
127.0.0.1:6899> XLEN s4
(integer) 0
```

**Note:** Your `XADD` will return a different ID, ensure you substitute it throughout.

From the above example you can see that the first `XDELEX ... ACKED` call returns `2`, because `grp` still has the message pending, and `XLEN` confirms it's still in the stream. Once `XACK` explicitly acknowledges it for `grp`, the same `XDELEX ... ACKED` call returns `1` and `XLEN` drops to `0` which means the message is only actually removed once **no** consumer group still needs it.

## `MOVE key db [ REPLACE ]` for moving database keys

Sometimes your application uses separate logical databases to represent different states of the same data, a staging area versus a live one for example, and it needs to promote a key from one to the other.

Before 9.2, `MOVE` could do that, but only if the destination key didn't already exist. If the key already existed in the destination, `MOVE` silently did nothing, leaving whatever was already there untouched. You'd have to copy the value to the destination yourself and then clean up the source, rather than letting `MOVE` handle the operation.

The new `REPLACE` option, added in 9.2.0, changes that. It tells `MOVE` to overwrite the key in the destination database if one is already present there.

Consider the following example:

```bash
127.0.0.1:6379> SELECT 0
OK
127.0.0.1:6379> SET session:42 "active"
OK
127.0.0.1:6379> SELECT 1
OK
127.0.0.1:6379[1]> SET session:42 "stale"
OK
127.0.0.1:6379[1]> SELECT 0
OK
127.0.0.1:6379> MOVE session:42 1
(integer) 0
127.0.0.1:6379> MOVE session:42 1 REPLACE
(integer) 1
127.0.0.1:6379> SELECT 1
OK
127.0.0.1:6379[1]> GET session:42
"active"
```

Here we have:

- Reply `1`: if key was moved
- Reply `0`: if key wasn't moved, either because it already exists in the destination database (and the `REPLACE` argument wasn't given), or because it doesn't exist in the source database

Without `REPLACE`, the first `MOVE` does nothing because `session:42` already exists in database 1. Adding `REPLACE` allows the move to continue, overwriting the `stale` value with the `active` value. For more information, see the [MOVE command documentation](https://valkey.io/commands/move/).

## `ZRANGE XX` for distinguishing missing keys from empty ranges

Sometimes your application needs to know if a range query came back empty because there was nothing there, or because the key you asked for wasn't there. Before 9.2, `ZRANGE` couldn't distinguish the difference between "this key doesn't exist" and "this key exists but has no members in the requested range".

A leaderboard that's empty because no one has scored yet, and a leaderboard that doesn't exist because you misspelled the key, both come back as the same empty array.

You could use `EXISTS leaderboard:weekly` before every `ZRANGE`, but that's a second round trip for information the server already has at the moment it runs the range query.

With Valkey 9.2, `ZRANGE` gains an `XX` option that surfaces that information directly in the reply:

```bash
127.0.0.1:6379> ZADD leaderboard:weekly 100 alice 85 bob
(integer) 2
127.0.0.1:6379> ZRANGE leaderboard:weekly 0 -1
1) "bob"
2) "alice"
127.0.0.1:6379> ZRANGE leaderboard:weekly 10 20
(empty array)
127.0.0.1:6379> ZRANGE leaderboard:monthly 0 -1
(empty array)
127.0.0.1:6379> ZRANGE leaderboard:weekly 10 20 XX
(empty array)
127.0.0.1:6379> ZRANGE leaderboard:monthly 0 -1 XX
(nil)
```

When the key exists, `XX` doesn't change the reply, even when the requested range is empty. When the key is missing, however, `XX` returns `(nil)` instead of an `(empty array)`. Your application can distinguish "the key exists but nothing matched" from "the key doesn't exist" without issuing a separate `EXISTS` call.

For more information, see the [ZRANGE command documentation](https://valkey.io/commands/zrange/).

## `SET IFNE` to set a key when its value doesn't match a specified value

Sometimes your application wants to update a value only if it hasn't already been set to something specific, to replace a stale default or placeholder, for example, without touching it if another process has already moved it on.

Before 9.2, doing this safely meant a `GET` first, checking the value in your application code, then conditionally issuing a `SET` for an extra round trip, and a race window between the `GET` and the `SET` where another client could write in between.

With Valkey 9.2, the `IFNE` option lets `SET` do that check itself. It sets a key only if the current value does **not** equal the comparison value, in a single call:

```bash
127.0.0.1:6379> SET foo hello
OK
127.0.0.1:6379> SET foo world IFNE hello
(nil)
127.0.0.1:6379> GET foo
"hello"
127.0.0.1:6379> SET foo world IFNE goodbye
OK
127.0.0.1:6379> GET foo
"world"
```

In this example, `SET foo world IFNE goodbye` means "Set foo to world, but only if foo's current value is **NOT** goodbye." Since `foo` is `"hello"`, which isn't `"goodbye"`, the condition passes and the write goes through.

In the earlier example, `SET foo world IFNE hello` had the opposite outcome. `foo` was already `"hello"`, so the condition failed, the write was skipped, and the reply came back `nil` instead of `OK`. There was no round trip to check the value first, and no window for another client to interleave.

For more information, see the [SET command](https://valkey.io/commands/set/).

## `SISMEMBER XX` for distinguishing missing keys from non-members

Sometimes your application needs to distinguish between "the user isn't in this set" and "the set doesn't exist."

Before 9.2, `SISMEMBER` couldn't distinguish the two cases by itself. For example, if you ran `SISMEMBER users:online alice` you'd get `0`. But what does `0` actually mean? That Alice isn't online? That `users:online` don't exist?

There is no way to tell. You could use `EXISTS users:online` then `SISMEMBER users:online alice` but now you've got another round trip operation, and your application needs to combine two pieces of information together.

With Valkey 9.2, `SISMEMBER` now has an `XX` option that makes that distinction explicit:

```bash
127.0.0.1:6379> SADD users:online alice bob
(integer) 2
127.0.0.1:6379> SISMEMBER users:online alice XX
(integer) 1
127.0.0.1:6379> SISMEMBER users:online carol XX
(integer) 0
127.0.0.1:6379> SISMEMBER users:offline alice XX
(integer) -1
```

Now your application can distinguish:

- `1` means Alice is a member
- `0` means the set exists, but the member isn't in it
- `-1` means the set doesn't exist

This lets applications distinguish missing state from a legitimate non-member result without issuing a separate `EXISTS` command.

For more information, see the [SISMEMBER command](https://valkey.io/commands/sismember/).

## Try it yourself

Whether you're locking on a version key, cleaning up a fan-out stream, or just tired of `EXISTS` round trips, these new and updated commands improve what application developers can reasonably do with Valkey.

If you're running an older version of Valkey, these examples are a good way to identify application logic that can become simpler after upgrading. [Clone the repository](https://github.com/valkey-io/valkey), check out the relevant branch or commit, build Valkey with `make`, and try the commands for yourself.

Have thoughts on any of these features? The [GitHub discussions](https://github.com/orgs/valkey-io/discussions) are open, and contributor feedback shapes what ships.
