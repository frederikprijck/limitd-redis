[![Build Status](https://travis-ci.org/auth0/limitd-redis.svg?branch=master)](https://travis-ci.org/auth0/limitd-redis)

`limitd-redis` is client for limits on top of `redis` using [Token Buckets](https://en.wikipedia.org/wiki/Token_bucket).
It's a fork from [LimitDB](https://github.com/limitd/limitdb).

## Installation

```
npm i limitd-redis
```

## Configure

Create an instance of `limitd-redis` as follows:

```js
const Limitd = require('limitd-redis');

const limitd = new Limitd({
  uri: 'localhost',
  //or
  nodes: [{
    port: 7000,
    host: 'localhost'
  }],
  buckets: {
    ip: {
      size: 10,
      per_second: 5
    }
  },
  prefix: 'test:',
  ping: {
    interval: 1000,
    maxFailedAttempts: 5,
    reconnectIfFailed: true
  }
});
```

### Options available:

- `uri` (string): Redis Connection String.
- `nodes` (array): [Redis Cluster Configuration](https://github.com/luin/ioredis#cluster).
- `buckets` (object): Setup your bucket types.
- `prefix` (string): Prefix keys in Redis.
- `ping` (object): Configure ping to Redis DB.

### Buckets:

- `size` (number): is the maximum content of the bucket. This is the maximum burst you allow.
- `per_interval` (number): is the amount of tokens that the bucket receive on every interval.
- `interval` (number): defines the interval in milliseconds.
- `unlimited` (boolean = false): unlimited requests (skip take).
- `skip_n_calls` (number): take will go to redis every `n` calls instead of going in every take.
- `elevated_limits` (object): elevated limits configuration that kicks in when the bucket is empty. Please refer to the [ERL section](#ERL-Elevated-Rate-Limits) for more details.

You can also define your rates using `per_second`, `per_minute`, `per_hour`, `per_day`. So `per_second: 1` is equivalent to `per_interval: 1, interval: 1000`.

If you omit `size`, limitdb assumes that `size` is the value of `per_interval`. So `size: 10, per_second: 10` is the same than `per_second: 10`.

If you don't specify a filling rate with `per_interval` or any other `per_x`, the bucket is fixed and you have to manually reset it using `PUT`.

### Ping:

- `interval` (number): represents the time between two consecutive pings. Default: 3000.
- `maxFailedAttempts` (number): is the allowed number of failed pings before declaring the connection as dead. Default: 5.
- `reconnectIfFailed` (boolean): indicates whether we should try to reconnect is the connection is declared dead. Default: true.



## Overrides
You can also define `overrides` inside your type definitions as follows:

```js
buckets = {
  ip: {
    size: 10,
    per_second: 5,
    overrides: {
      '127.0.0.1': {
        size: 100,
        per_second: 50
      }
    }
  }
}
```

In this case the specific bucket for `127.0.0.1` of type `ip` will have a greater limit.

It is also possible to define overrides by regex:

```js
overrides: {
  'local-ips': {
    match:      /192\.168\./
    size:       100,
    per_second: 50
  }
}
```

It's possible to configure expiration of overrides:

```js
overrides: {
  '54.32.12.31': {
    size:       100,
    per_second: 50,
    until:      new Date(2016, 4, 1)
  }
}
```

## ERL (Elevated Rate Limits)
### Prerequisites
Redis 6.2+ is required to use ERL.

### Introduction
ERL is a feature that allows you to define a different set of limits that kick in when the bucket is empty.
The feature aims to provide a way to temporarily allow a higher rate of requests when the bucket is empty, for a limited period of time.

To be able to allow its use within limitd-redis, you need to:
1. call the `takeElevated` method.
2. pass the `elevated_limits` parameter with the following properties:
   - `erl_is_active_key`: the identifier of the ERL activation for the bucket. This works similarly to the `key` you pass to `limitd.take`, which is the identifier of the bucket; however it's used to track the ERL activation for the bucket instead
   - `erl_quota_key`: the identifier of the ERL quota bucket name.
3. make sure that the bucket definition has ERL configured.

### Configuration
You can configure elevated limits inside your bucket definitions as follows:

```js
buckets = {
  ip: {
    size: 10,
    per_second: 5,
    elevated_limits: { // Optional. ERL configuration if needed for the bucket. If not defined, the bucket will not use ERL.
      size: 100, // Optional. New bucket size. already used tokens will be deducted from current bucket content upon ERL activation. Default: same as the original bucket.
      per_second: 50, // Optional. New bucket refill rate. You can use all the other refill rate configurations defined above, such as per_minute, per_hour, per_interval etc. Default: same as the original bucket.
    }
  }
}
```

### ERL Quota
ERL quota represents the number of ERL activations that can be performed in a calendar month for the given `erl_quota_key`.

When ERL is triggered, it will keep activated for the `erl_activation_period_seconds` defined in the bucket configuration.

The amount of minutes per month allowed in ERL mode is defined by: `quota_per_calendar_month * erl_activation_period_seconds / 60`.

The overrides in ERL work the same way as for the regular bucket. Both size and per_interval are mandatory when specifying an override. 

## Breaking changes from `Limitdb`

* Elements will have a default TTL of a week unless specified otherwise.

## TAKE

```js
limitd.take(type, key, { count, configOverride }, (err, result) => {
  console.log(result);
});
```

`limitd.take` takes the following arguments:

-  `type`: the bucket type.
-  `key`: the identifier of the bucket.
-  `count`: the amount of tokens you need. This is optional and the default is 1.
-  `configOverride`: caller-provided bucket configuration for this operation

The result object has:
-  `conformant` (boolean): true if the requested amount is conformant to the limit.
-  `remaining` (int): the amount of remaining tokens in the bucket.
-  `reset` (int / unix timestamp): unix timestamp of the date when the bucket will be full again.
-  `limit` (int): the size of the bucket.

## TAKEELEVATED

This take operation allows the use of elevated rate limits if it corresponds.

```js
limitd.takeElevated(type, key, { count, configOverride, elevated_limits }, (err, result) => {
  console.log(result);
});
```

`limitd.takeElevated` takes the following arguments:

-  `type`: the bucket type.
-  `key`: the identifier of the bucket.
-  `count`: the amount of tokens you need. This is optional and the default is 1.
-  `configOverride`: caller-provided bucket configuration for this operation
-  `elevated_limits`: (object)
  - `erl_is_active_key`: (string) the identifier of the ERL activation for the bucket.
  - `erl_quota_key`: (string) the identifier of the ERL quota bucket name.
  - `erl_activation_period_seconds`: (int) the ERL activation period as defined in the bucket configuration used in the current request.
  - `quota_per_calendar_month`: (int) the amount of ERL activations that can be done in a calendar month. Each activation will remain active during `erl_activation_period_seconds`.

`quota_per_calendar_month` is the only refill rate available for ERL quota buckets at the moment. 
The quota bucket will be used to track the amount of ERL activations that can be done in a calendar month. 
If the quota bucket is empty, the ERL activation will not be possible. 
The quota bucket will be refilled at the beginning of every calendar month.

For instance, if you want to allow a user to activate ERL for a bucket only 5 times in a month, you can define a quota bucket with `quota_per_calendar_month: 5`.
That means that the user can activate ERL for the bucket 5 times in a month, and after that, the ERL activation will not be possible until the start of the next month.
The total minutes allowed for ERL activation in a calendar month is calculated as follows: `quota_per_calendar_month * erl_activation_period_seconds / 60`.

The result object has:
-  `conformant` (boolean): true if the requested amount is conformant to the limit.
-  `remaining` (int): the amount of remaining tokens in the bucket.
-  `reset` (int / unix timestamp): unix timestamp of the date when the bucket will be full again.
-  `limit` (int): the size of the bucket.
-  `elevated_limits` (object)
  -  `triggered` (boolean): true if ERL was triggered in the current request.
  -  `activated` (boolean): true if ERL is activated. Not necessarily triggered in this call.
  -  `quota_remaining` (int): **[Only valid if triggered=true]** If `triggered=true`, this value contains the remaining quota count for the given `erl_quota_key`. Otherwise, it will return -1, which is not valid to be interpreted as a quota count.
  -  `quota_allocated`: (int): amount of quota allocated in the bucket configuration. This value is defined in the bucket configuration and is the same as `quota_per_calendar_month`.
  -  `erl_activation_period_seconds`: (int): the ERL activation period as defined in the bucket configuration used in the current request.

Example of interpretation:
``` javascript
if erl_triggered // quota left in the quotaKey bucket
if !erl_triggered // ERL wasn't triggered in this call, so we haven't identified the remaining quota.
```

## PUT

You can manually reset a fill a bucket using PUT:

```js
limitd.put(type, key, [count], (err, result) => {
  console.log(result);
});
```

`limitd.put` takes the following arguments:

-  `type`: the bucket type.
-  `key`: the identifier of the bucket.
-  `count`: the amount of tokens you want to put in the bucket. This is optional and the default is the size of the bucket.
-  `configOverride`: caller-provided bucket configuration for this operation

## Overriding Configuration at Runtime
Since the method of storing overrides for buckets in memory does not scale to a large number, limitd-redis provides a way for callers to pass in configuration from an external data store.  The shape of this `configOverride` parameter (available on `take`, `put`, `get`, and `wait`) is exactly the same as `Buckets` above ^.

An example configuration override call might look like this:

```js
const configOverride = {
  size: 45,
  per_hour: 15
}
// take one
limitd.take(type, key, { configOverride }, (err, result) => {
  console.log(result);
});
// take multiple
limitd.take(type, key, { count: 3, configOverride }, (err, result) => {
  console.log(result);
});
```

Config overrides follow the same rules as Bucket configuration elements with respect to default size when not provided and ttl.

### Overriding Configuration at Runtime with ERL
We can also override the configuration for ERL buckets at runtime. The shape of this `configOverride` parameter is the same as `Buckets` above.

An example configuration override call for ERL might look like this:

```js
const configOverride = {
  size: 45,
  per_hour: 15,
  elevated_limits: {
    size: 100,
    per_hour: 50,
  }
}
```

## Author

[Auth0](auth0.com)

## License

This project is licensed under the MIT license. See the [LICENSE](LICENSE) file for more info.
