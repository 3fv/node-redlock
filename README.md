[![npm version](https://badge.fury.io/js/redlock.svg)](https://www.npmjs.com/package/redlock)
[![Build Status](https://travis-ci.org/mike-marcacci/node-redlock.svg)](https://travis-ci.org/mike-marcacci/node-redlock)
[![Coverage Status](https://coveralls.io/repos/mike-marcacci/node-redlock/badge.svg)](https://coveralls.io/r/mike-marcacci/node-redlock)

# Redlock

This is a node.js implementation of the [redlock](http://redis.io/topics/distlock) algorithm for distributed redis locks. It provides strong guarantees in both single-redis and multi-redis environments, and provides fault tolerance through use of multiple independent redis instances or clusters.

- [Installation](#installation)
- [Usage (Promise Style)](#usage-promise-style)
- [Usage (Disposer Style)](#usage-disposer-style)
- [Usage (Callback Style)](#usage-callback-style)
- [Locking multiple resources](#locking-multiple-resources)
- [API Docs](#api-docs)

### High-Availability Recommendations

- Use at least 3 independent servers or clusters
- Use an odd number of independent redis **_servers_** for most installations
- Use an odd number of independent redis **_clusters_** for massive installations
- When possible, distribute redis nodes across different physical machines

### Using Cluster/Sentinel

**_Please make sure to use a client with built-in cluster support, such as [ioredis](https://github.com/luin/ioredis)._**

It is completely possible to use a _single_ redis cluster or sentinal configuration by passing one preconfigured client to redlock. While you do gain high availability and vastly increased throughput under this scheme, the failure modes are a bit different, and it becomes theoretically possible that a lock is acquired twice:

Assume you are using eventually-consistent redis replication, and you acquire a lock for a resource. Immediately after acquiring your lock, the redis master for that shard crashes. Redis does its thing and fails over to the slave which hasn't yet synced your lock. If another process attempts to acquire a lock for the same resource, it will succeed!

This is why redlock allows you to specify multiple independent nodes/clusters: by requiring consensus between them, we can safely take out or fail-over a minority of nodes without invalidating active locks.

To learn more about the the algorithm, check out the [redis distlock page](http://redis.io/topics/distlock).

### How do I check if something is locked?

The purpose of redlock is to provide exclusivity guarantees on a resource over a duration of time, and is not designed to report the ownership status of a resource. For example, if you are on the smaller side of a network partition you will fail to acquire a lock, but you don't know if the lock exists on the other side; all you know is that you can't guarantee exclusivity on yours. This is further complicated by retry behavior, and even moreso when acquiring a lock on more than one resource.

That said, for many tasks it's sufficient to attempt a lock with `retryCount=0`, and treat a failure as the resource being "locked" or (more correctly) "unavailable".

Note that with `retryCount=-1` there will be unlimited retries until the lock is aquired.

## Installation

```bash
npm install --save redlock
```

## Configuration

Redlock is designed to use [ioredis](https://github.com/luin/ioredis) to keep its client connections and handle the cluster protocols.

A redlock object is instantiated with an array of at least one redis client and an optional `options` object. Properties of the Redlock object should NOT be changed after it is first used, as doing so could have unintended consequences for live locks.

```ts
import Client from "ioredis";
import Redlock from "./redlock";

const redisA = new Client({ host: "a.redis.example.com" });
const redisB = new Client({ host: "b.redis.example.com" });
const redisC = new Client({ host: "c.redis.example.com" });

const redlock = new Redlock(
  // You should have one client for each independent redis node
  // or cluster.
  [redisA, redisB, redisC],
  {
    // The expected clock drift; for more details see:
    // http://redis.io/topics/distlock
    driftFactor: 0.01, // multiplied by lock ttl to determine drift time

    // The max number of times Redlock will attempt to lock a resource
    // before erroring.
    retryCount: 10,

    // the time in ms between attempts
    retryDelay: 200, // time in ms

    // the max time in ms randomly added to retries
    // to improve performance under high contention
    // see https://www.awsarchitectureblog.com/2015/03/backoff.html
    retryJitter: 200, // time in ms
  }
);
```

## Error Handling

Because redlock is designed for high availability, it does not care if a minority of redis instances/clusters fail at an operation.

However, it can be helpful to monitor and log such cases. Redlock emits an "error" event whenever it encounters an error, even if the error is ignored in its normal operation.

```ts
redlock.on("error", (error) => {
  // Ignore cases where a resource is explicitly marked as locked on a client.
  if (error instanceof ResourceLockedError) {
    return;
  }

  // Log all other errors.
  console.error(error);
});
```

Additionally, a per-attempt and per-client stats (including errors) are made available on the `attempt` propert of both `Lock` and `ExecutionError` classes.

## Usage

The `using` method wraps and executes a routine in the context of an auto-extending lock, returning a promise of the routine's value. In the case that auto-extension fails, an AbortSignal will be updated to indicate that abortion of the routine is in order, and to pass along the encountered error.

```ts
await redlock.using([senderId, recipientId], 5000, async (signal) => {
  // Do something...
  await something();

  // Make sure any necessary lock extension has not failed.
  if (signal.aborted) {
    throw signal.error;
  }

  // Do something else...
  await somethingElse();
});
```

Alternatively, locks can be acquired and released directly:

```ts
// Acquire a lock.
let lock = await redlock.acquire(["a"], 5000);

// Do something...
await something();

// Extend the lock.
lock = await lock.extend(5000);

// Do something else...
await somethingElse();

// Release the lock.
await lock.release();
```
