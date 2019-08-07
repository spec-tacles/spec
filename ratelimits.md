# Spectacles ratelimiting specification

Details about how Spectacles controls REST ratelimits received from Discord.

## Redis

Ratelimits are stored in Redis with 2 keys for each ratelimit bucket and 1 key for global ratelimits.

| Key | Expiration | Default | Description |
|-------------------|----------------------------------|---------|----------------------------------------------------------------------------------------------------------------------------------------------------------------|
| global | When the global ratelimit resets |  | Whether the application is globally ratelimited; the value is irrelevant. |
| \[route\]:remaining | When the bucket resets | 0 | The total number of remaining requests in this bucket. This value is NOT derived from Discord; instead it is automatically decremented when a request is made. |
| \[route\]:limit | None | 1 | The total number of requests that can be made after the bucket resets. This value is directly stored from Discord headers. |

The calculation to determine the timeout for a given bucket is as follows, where the keys are `[route]:remaining`, `[route]:limit`, and `global` and the argument is a default timeout when one cannot be determined (recommended 1e3ms):

```lua
local global = redis.call('pttl', KEYS[3])
if global > 0 then return global end

local remaining = tonumber(redis.call('get', KEYS[1]))

if remaining == nil then
	local limit = tonumber(redis.call('get', KEYS[2]))
	if limit == nil then limit = 1 end

	redis.call('set', KEYS[1], limit - 1)
	return 0
end

local ttl = redis.call('pttl', KEYS[1])
if remaining <= 0 then
	if ttl < 0 then return tonumber(ARGV[1])
	else return ttl end
end

redis.call('set', KEYS[1], remaining - 1)
if ttl > 0 then redis.call('pexpire', KEYS[1], ttl) end
return 0
```

All duration values must be given and received as milliseconds.

Clients must use the value from this script tentatively and attempt to re-claim a request in the bucket once the timeout expires. Only after a client receives a 0 value from this script can it then complete the queued request.

Once a client receives ratelimit headers from Discord, it must ONLY set:

- The expiration for the `global` key, if any
- The expiration for the `[route]:remaining` key
- The `[route]:limit` key

Expirations should be determined from headers (`X-Ratelimit-Reset` - `Date` = seconds to timeout). In the case of a 429 status code from Discord, clients should prefer the `Reset-After` header (given in milliseconds).
