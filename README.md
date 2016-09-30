# lru-cache-for-clusters-as-promised

[![lru-cache-for-clusters-as-promised](https://img.shields.io/npm/v/lru-cache-for-clusters-as-promised.svg)](https://www.npmjs.com/package/lru-cache-for-clusters-as-promised)
![Build Status](https://jenkins.doublesharp.com/badges/build/lru-cache-for-clusters-as-promised.svg)
![Code Coverage](https://jenkins.doublesharp.com/badges/coverage/lru-cache-for-clusters-as-promised.svg)
[![Code Climate](https://codeclimate.com/github/doublesharp/lru-cache-for-clusters-as-promised/badges/gpa.svg)](https://codeclimate.com/github/doublesharp/lru-cache-for-clusters-as-promised)
![Dependency Status](https://david-dm.org/doublesharp/lru-cache-for-clusters-as-promised.svg)
![Dev Dependency Status](https://david-dm.org/doublesharp/lru-cache-for-clusters-as-promised/dev-status.svg)
![Downloads](https://img.shields.io/npm/dt/lru-cache-for-clusters-as-promised.svg)

LRU Cache for Clusters as Promised provides a cluster-safe [`lru-cache`](https://www.npmjs.com/package/lru-cache) via Promises. For environments not using [`cluster`](https://nodejs.org/api/cluster.html), the class will provide a Promisified interface to a standard [`lru-cache`](https://www.npmjs.com/package/lru-cache).

Each time you call [`cluster.fork()`](https://nodejs.org/api/cluster.html#cluster_cluster_fork_env), a new thread is spawned to run your application. When using a load balancer even if a user is assigned a particular IP and port these values are shared between the [`workers`](https://nodejs.org/api/cluster.html#cluster_class_worker) in your cluster, which means there is no guarantee that the user will use the same `workers` between requests. Caching the same objects in multiple threads is not an efficient use of memory. 

LRU Cache for Clusters as Promised stores a single `lru-cache` on the [`master`](https://nodejs.org/api/cluster.html#cluster_cluster_ismaster) thread which is accessed by the `workers` via IPC messages. The same `lru-cache` is shared between `workers` having a common `master`, so no memory is wasted.

# install
```shell
npm install --save lru-cache-for-clusters-as-promised
```

# options

* `namespace: string`, default `"default"`;
  * The namespace for this cache on the master thread as it is not aware of the worker instances.
* `timeout: integer`, default `100`.
  * The amount of time in milliseconds that a worker will wait for a response from the master before rejecting the `Promise`.
* `failsafe: string`, default `resolve`.
  * When a request times out the `Promise` will return `resolve(undefined)` by default, or with a value of `reject` the return will be `reject(Error)`.
* `max: number`
  * The maximum items that can be stored in the cache
* `maxAge: milliseconds`
  * The maximum age for an item to be considered valid
* `stale: true|false`
  * When `true` expired items are return before they are removed rather than `undefined`

> ! note that `length` and `dispose` are missing as it is not possible to pass `functions` via IPC messages.

# api

* `set(key, value)`
  * Sets a value for a key.
* `get(key)`
  * Returns a value for a key.
* `peek(key)`
  * Returns the value for a key without updating its last access time.
* `del(key)`
  * Removes a value from the cache.
* `has(key)`
  * Returns true if the key exists in the cache.
* `incr(key, [amount])`
  * Increments a numeric key value by the `amount`, which defaults to `1`. More atomic in a clustered environment.
* `decr(key, [amount])`
  * Decrements a numeric key value by the `amount`, which defaults to `1`. More atomic in a clustered environment.
* `reset()`
  * Removes all values from the cache.
* `keys()`
  * Returns an array of all the cache keys.
* `values()`
  * Returns an array of all the cache values.
* `dump()`
  * Returns a serialized array of the cache contents.
* `prune()`
  * Manually removes items from the cache rather than on get.
* `length()`
  * Return the number of items in the cache.
* `itemCount()`
  * Return the number of items in the cache - same as `length()`.

# example usage
```javascript
// require the module in your master thread that creates workers to initialize
const LRUCache = require('lru-cache-for-clusters-as-promised');

LRUCache.init();
```

```javascript
// worker code
const LRUCache = require('lru-cache-for-clusters-as-promised');
const cache = new LRUCache({
  namespace: 'users',
  max: 50,
  stale: false,
  timeout: 100,
  failsafe: 'resolve',
});

const user = { name: 'user name' };
const key = 'userKey';

// set a user for a the key
cache.set(key, user)
.then(() => {
  console.log('set the user to the cache');

  // get the same user back out of the cache
  return cache.get(key);
})
.then((cachedUser) => {
  console.log('got the user from cache', cachedUser);

  // check the number of users in the cache
  return cache.length();
})
.then((size) => {
  console.log('user cache size/length', size);

  // remove all the items from the cache
  return cache.reset();
})
.then(() => {
  console.log('the user cache is empty');

  // return user count, this will return the same value as calling length()
  return cache.itemCount();
})
.then((size) => {
  console.log('user cache size/itemCount', size);
});

```

# process flow

**Clustered cache on master thread for clustered environments**
```
                                                                +-----+
+--------+  +---------------+  +---------+  +---------------+   # M T #
|        +--> LRU Cache for +-->         +--> Worker Sends  +--># A H #
| Worker |  |  Clusters as  |  | Promise |  |  IPC Message  |   # S R #
|        <--+   Promised    <--+         <--+   to Master   <---# T E #
+--------+  +---------------+  +---------+  +---------------+   # E A #
                                                                # R D #
v---------------------------------------------------------------+-----+
+-----+
* W T *   +--------------+  +--------+  +-----------+
* O H *--->   Master     +-->        +--> LRU Cache |
* R R *   | IPC Message  |  | Master |  |    by     |
* K E *<--+  Listener    <--+        <--+ namespace |
* E A *   +--------------+  +--------+  +-----------+
* R D *
+-----+
```

**Promisified for non-clustered environments**
```
+---------------+  +---------------+  +---------+  +-----------+
|               +--> LRU Cache for +-->         +-->           |
| Non-clustered |  |  Clusters as  |  | Promise |  | LRU Cache |
|               <--+   Promised    <--+         <--+           |
+---------------+  +---------------+  +---------+  +-----------+
```
