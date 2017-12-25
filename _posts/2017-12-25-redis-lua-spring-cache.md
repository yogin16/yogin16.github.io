---
layout:     post
title:      "Clean Java's Spring's Redis Cache with a Lua script"
date:       2017-12-25 20:04:20 +0530
comments:   true
---

[Lua](http://www.lua.org/manual/5.1/manual.html) is an old programming language developed with motive to be embedded in other applications like Redis. The idea of lua script in redis in general is very useful. One can "cache" the script in redis server to execute series of instructions from just one EVAL command from client.

Redis lets us write scripted extensions and easy interface to manage the script. [This guide](https://www.compose.com/articles/a-quick-guide-to-redis-lua-scripting/) describes the basics of Lua scripting in Redis pretty neatly. I wanted to have a lua script for Redis to learn more about basic Lua itself.

We use redis for Spring's `@Cacheable` cache mechanism. The usage is pretty simple; on any @Component (Service) bean if we want to cache the result of the method we just annotate with:

```java
    @Cacheable(value = "A_METHOD", key = "#param")
    public Response aMethod(String param) {
        //implementation - usually loads data for param from db
        return response;
    }
```

To evict the cache we have `@CacheEvict` - for more details in the spring caching one can follow [this link](http://www.baeldung.com/spring-cache-tutorial).

Goal in this post is to evict/clear the cache `A_METHOD` directly in redis. (Cases where we have changed the entry in the db directly and the `@CacheEvict` from spring was not called; making the redis cache stale)

### Spring's Cache
Spring utilizes Redis's ZSET for managing the cache keys. [ZADD](https://redis.io/commands/zadd) and [ZREVRANGE](https://redis.io/commands/zrevrange) operations would allow to manage the zset.

Based on our `@Cacheable` definition the zset name maintaining keys on the redis would be `A_METHOD~keys`. If we connect to redis server from [shell](https://redis.io/topics/rediscli) with `./redis-cli` we can access the zset something like following:
```bash
    $ zrevrange A_METHOD~keys 0 100
    1) "param1"
    2) "param2"
    3) "param4"
```

Each keys, in this case, `param1`, `param2`, `param4` are again redis keys of simple [set](https://redis.io/commands/set) . We can access Response for each keys by [get](https://redis.io/commands/get).
We want to clear this cache.

### Clearing zset entries
This is basically two step process:
- Remove key from zset: operation [zrem](https://redis.io/commands/zrem), would remove the key `param1` from `A_METHOD~keys` zset.
- Delete the key value pair: operation [del](https://redis.io/commands/del), would delete the value associated to the `param1` key

We need to do above for all keys in the zset to clear the whole cache.
I was able to come up with following lua script to that:

<script src="https://gist.github.com/yogin16/931a354933e41c14fca9f6113497db20.js"></script>

To execute the script we can use shell as this:
```bash
$ ./redis-cli --eval cleanSpringRedisCache.lua A_METHOD
```

Ability to add scripts in redis is a huge win in terms of efficiency and productivity. Multiple APIs for scripting provided by redis like `SCRIPT LOAD` and `EVALSHA` ensures that we don't have to upload the script to the server everytime we have to run it. And we can give multiple instruction or set of commands for redis in one script for doing bulk operations. We can also use many of existing Lua's libraries. That saves the need to for taking lot of Redis connections for each individual operations from clients and also network overhead.

## References:
1. https://www.redisgreen.net/blog/intro-to-lua-for-redis-programmers/
1. https://www.compose.com/articles/a-quick-guide-to-redis-lua-scripting/
1. https://redislabs.com/blog/5-6-7-methods-for-tracing-and-debugging-redis-lua-scripts/
1. https://redis.io/commands
1. http://www.baeldung.com/spring-cache-tutorial
1. http://blog.trifork.com/2015/02/09/active-cache-eviction-with-ehcache-and-spring-framework/
1. http://www.lua.org/manual/5.1/manual.html