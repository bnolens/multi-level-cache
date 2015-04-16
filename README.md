# Multi Level Cache

Multi Level Cache allows you to manage a local and remote cache with a single API/module.

[![Build Status](https://travis-ci.org/guyellis/multi-level-cache.svg?branch=master)](https://travis-ci.org/guyellis/multi-level-cache)
[![Code Climate](https://codeclimate.com/github/guyellis/multi-level-cache/badges/gpa.svg)](https://codeclimate.com/github/guyellis/multi-level-cache)
[![Dependency Status](https://david-dm.org/guyellis/multi-level-cache.png)](https://david-dm.org/guyellis/multi-level-cache)

# Install:

```
npm install multi-level-cache [--save]
```

# Usage:

```
var MultiCache = require('multi-level-cache');
var multiCache = new MultiCache('node-cache', 'redis');

multiCache.set('myKey', 'myValue', function(err, result) {
  // Key/Value has been set in both the local in-memory node-cache
  // and in the remote redis cache
});

multiCache.get('myKey', function(err, result) {
  // value is now set to {myKey: 'myValue'}
  // By default it will look in local cache first and then remote
  // cache if not found locally.
});
```

# API

## var multiCache = new MultiCache(localCache, remoteCache [,options])

* `localCache`
  * String, such as 'node-cache', representing a cache known by the system
  to use as the local cache.
  * Object/function that exposes the methods `set`, `get`, `del`
* `remoteCache`
  * String, such as 'redis', representing a cache known by the system
  to use as the remote cache.
  * Object/function that exposes the methods `set`, `get`, `del`
* `options`
  * `useLocalCache` - if set to a truthy value then by default the local
  cache will be active for all operations. If missing then will default
  to true.
  * `useRemoteCache` - if set to a truthy value then by default the remote
  cache will be active for all operations. If missing then will default
  to true.
  * `localOptions` - if a string is passed in for the localCache then set
  `localOptions` to an object to be used when creating the local cache.
  * `remoteOptions` - if a string is passed in for the remoteCache then set
  `remoteOptions` to an object to be used when creating the remote cache.

## `set(key, value[, ttl, options, callback])`

* `key`
  * The key to use in the cache
* `value`
  * The value to be stored against that key
* `ttl`
  * The time-to-live for that cache object in seconds.
* `options`
  * `useLocalCache` - allows the overriding of the default value for local
  cache use on an operation-by-operation basis.
  * `useRemoteCache` - allows the overriding of the default value for remote
  cache use on an operation-by-operation basis.
* `callback`
  * A callback function that will be called with an `error` as the first
  parameter (if there is one) and the value as the second parameter.
  * `callback(err, value)`

## `get(keys, [options,] callback)`

* `keys`
  * The keys to find in the cache
* `options`
  * `useLocalCache` - allows the overriding of the default value for local
  cache use on an operation-by-operation basis.
  * `useRemoteCache` - allows the overriding of the default value for remote
  cache use on an operation-by-operation basis.
  * `setLocal` - If the local cache is empty and the keys are found in the 
  remote cache and `setLocal` is truthy then the local cache will be updated
  with the results from the remote cache.
* `callback`
  * A callback function that will be called with an `error` as the first
  parameter (if there is one) and the value as the second parameter.
  * `callback(err, value)`

## `del(keys, [options,] callback)`

* `keys`
  * The keys to remove from the cache(s).
* `options`
  * `useLocalCache` - allows the overriding of the default value for local
  cache use on an operation-by-operation basis.
  * `useRemoteCache` - allows the overriding of the default value for remote
  cache use on an operation-by-operation basis.
* `callback`
  * A callback function that will be called with an `error` as the single
  parameter (if there is one).
  * `callback(err)`

# Details

The Multi Level Cache module does not actually do any caching itself.
It relies on other modules to do the caching and it acts as a way to efficiently manage 
a local and remote cache as a single unit.

Adapters are currently provided for the following cache modules:

* [node-cache](https://github.com/tcs-de/nodecache)
* [redis](https://github.com/mranney/node_redis)

If there is a cache module that you want to use with the Multi Level Cache that's
not listed above then [add a an issue](https://github.com/guyellis/multi-level-cache/issues) or
a Pull Request to get it in there.

Here are some ways that you might use this module. The use cases assume that you're running
a load-balanced cluster of servers. Many of these use cases will also apply to single server
instances that want to maintain a local cache backed by a remote cache for fast start times
and redundancy or memory efficiency (LRU) in the local cache.

## Use Case: Immutable data from a slow source

Need: Your application needs to retrieve immutable data from a source that is slow. Once you've
retrieved the data you keep it in local memory because it's immutable and is not going to change.

Use:

* You setup the cache and issue a `get()` on the key.
* Both the local and remote caches are checked and do not find the data.
* You retrieve the data from the slow source and do a `set()` which places it in both caches.
* The next time this server needs that data it will do a `get()` and check the local cache first.
  It finds the data in the local cache and returns that will no network request.
* Another server in the cluster needs this data. It does a `get()` on the cache for the data. It
  first checks the local cache but that is empty. The remote cache, however, has the data and returns
  it bypassing the need to go to the slow source. Because the `setLocal` flag has been set it also
  updates the local cache with the data so that when this server makes that request again the local
  cache is able to service the request.
  
# Use Case: Time insensitive data
  
Need: You have data that is built by making several (possibly expensive) DB calls and processing
that data with your business rules. Once the data has been assembled it's good for a given period
of time because the underlying data rarely changes. It's also okay if the server are out-of-sync
for a short period of time.

Use:

* Check to see if you already have the data available by calling the `get()` method on the 
multi-level-cache with the `setLocal` option set to true. `setLocal` will update the local
cache if the data is only found in the remote cache.
* If there's nothing in the cache then assemble it from DB calls and processing and use `set()` to
put it in the caches. When using `set()` specify a TTL of (say) 1 minute.
* Depending on the size of your cluster of servers and the number of DB calls you make this will 
reduce your application's impact on DB resources by only making requests to the DB around a maximum
of once per minute (for our example) while still keeping all servers relatively up-to-date by using
a combination of a local and remote cache.

# Use Case: Rate limiting

Need: Your application makes calls out to social media applications to use data from their APIs.
The social media applications have rate limiting associated with the token that you're using or
the IPs from which you are making the calls.

Use:

* Check your local last-updated-time to see if it's time to fetch more data from the source.
* If it is then check the last-updated-time in the remote cache to see if it is current or not.
* If it is current then update your local cache with the remote cache.
* If it is not then call the API to update both local and remote cache.
* Other servers in the cluster will get the benefit of this server updating the remote cache
when their local last-updated-time expires.
* Your calls to the API have reduced and aleviated your rate limit problem.

