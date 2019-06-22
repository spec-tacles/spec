# Spectacles Config

All Spectacles applications use a single config file format and structure to simplify the
configuration process across multiple Spectacles services.

[TOML](https://github.com/toml-lang/toml) version: `0.5.0`

```toml
[discord]
token = "" # your Discord bot token
	[discord.shards]
	count = 1 # the number of shards you will spawn (optional)
	ids = [0] # the specific shard IDs you will spawn here (optional)

[broker]
url = "" # the URL (including protocol) of the message broker
	[broker.groups]
	gateway = "" # the broker group to use with the gateway
```

All other configuration formats (including environment variables) are non-compliant.
