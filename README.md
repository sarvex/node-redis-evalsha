# shavaluator-js

This library provides a convenient wrapper for sending Lua scripts to a Redis server via `EVALSHA`.

It works in tandem with [mranney/node-redis](https://github.com/mranney/node_redis). Note that
node-redis already tries to use evalsha if you use eval. This library merely prevents your code
from computing a SHA1 every time you execute a script.

#### What is EVALSHA?

`EVALSHA` allows you to send Lua scripts to a Redis server by sending the SHA-1 hashes instead of actual script content. As long as the body of your script was previously sent to Redis via `EVAL` or `SCRIPT LOAD`, you can use `EVALSHA` to avoid the overhead of sending your entire Lua script over the network.

A `Shavaluator` object wraps a Redis client for executing Lua scripts. When executing Lua scripts, a shavaluator will always attempt `EVALSHA` first, falling back on `EVAL` if the script has not yet been cached by the Redis server.

This project was forked from [jeffomatic/shavaluator-js](https://github.com/jeffomatic/shavaluator-js)
for these reasons:

   * simplify the library -
     23 files changed, 461 insertions(+), 1211 deletions(-)
   * coffee-script is dumb
   * use redis `sendCommand` instead of `eval` method as it tries to be too
     smart and check the sha that we already computed.
   * ability to use same instance with multiple redis instances

#### Example

```js
var Shavaluator = require('redis-evalsha')

// 1. Initialize a shavaluator with a Redis client
var shavaluator = new Shavaluator(redis);

// 2. Add a series of named Lua scripts to the shavaluator.
shavaluator.add('delequal',
    "if redis.call('GET', KEYS[1]) == ARGV[1] then\n" +
    "  return redis.call('DEL', KEYS[i])\n" +
    "end\n" +
    "return 0\n");

// 3. The 'delequal' script is now available to call using `exec`. When you
//    call this, first EVALSHA is attempted, and then it falls back to EVAL.
shavaluator.exec('delequal', ['someKey'], ['deleteMe'], function(err, result) {
  console.log(err, result);
});
```

### Adding scripts

Before you can run Lua scripts, you should give each one a name and add them to a shavaluator.

```js
scripts = {
  delequal:
    " \
    if redis.call('GET', KEYS[1]) == ARGV[1] then \
      return redis.call('DEL', KEYS[i]) \
    end \
    return 0 \
    "

  zmembers:
    " \
    local key = KEYS[1] \
    local results = {} \
    if redis.call('ZCARD', key) == 0 then \
      return {} \
    end \
    for i = 1, #ARGV, 1 do \
      local memberName = ARGV[i] \
      if redis.call('ZSCORE', key, memberName) then \
        table.insert(results, memberName) \
      end \
    end \
    return results;
    "
};

for (var name in scripts) {
  shavaluator.add(name, scripts[name]);
}
```

Adding a script only generates the SHA-1 of the script body; it **does not**
perform any network operations.

## Class reference

### constructor(redisClient)

### add(name, body)

Adds a Lua script to the shavaluator.

### exec(scriptName, keysArray, argsArray, callback)

Executes the script named `scriptName`.

The `callback` parameter is standard asynchronous callback, taking two arguments:

1. an error, which is null on success
2. the script result
