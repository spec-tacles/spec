# Spectacles

Spectacles is a communication protocol and library ecosystem for coordinating microservices. Each
application is built around a central message broker that coordinates messaging between services.

## Service Architecture

Each service has 2 properties that help it communicate with other services.

1. Group name
2. Client name

### Group Name

Services identify themselves by a globally unique group name that is shared amongst all instances
of that service but is guaranteed to never be used by any other instance. This is usually a build-
time constant, since a service should never change its identity during runtime.

For example, a service dedicated to handling incoming HTTP requests could have a group name of
`INCOMING_HTTP`.

Every message is delivered at most once to a single client in every group.

### Client Name

Each instance of each service also identifies itself by a globally unique name. This name is
usually generated randomly at runtime.

## Communication System

Every message broker has at least 3 concepts:

1. Publish
2. Call
4. Consume

Every message emitted by the message broker has at least 2 concepts:

1. Acknowledge
2. Reply

### Publish

When an event is generated and a response is not required, the client which created the event
will "publish" it to the other clients. Each event has at least 2 attributes:

1. Name
2. Data

The event name is required to enable the receiving app to distinguish what data it should expect.

Publish must return the unique ID of this event emission.

### Call

Call behaves similarly to publish, except the calling client will wait for a single response from
another client.

### Consume

When a client consumes, it signals an intent to receive events for its group. The client must
specify which events it intends to receive.

### Acknowledge

When a message is acknowledged, it is removed from pending state and will no longer be re-delivered
to other consumers if the recipient client dies or times out.

### Reply

When a client receives a "publish" event, it can choose to respond; if the source client is
listening, it will receive the response.

## Message Brokers

Supported message brokers.

### Redis

Specatcles' primary method of communication happens over Redis streams & pubsub. Most communication
happens via streams, but pubsub is leveraged to simplify RPC responses.

#### Publish

- `XADD {event} * data {data}`

#### Call

- Publish
- `SUBSCRIBE {event}:{publish_id}`

#### Consume

Due to Redis streams, the consumption process is two-fold. Each client must:

1. Periodically run `XAUTOCLAIM` to ensure that all orphaned messages are handled
2. Use `XREADGROUP` to consume messages for its group

Both consumption processes emit events for the client to handle.

##### XAUTOCLAIM

Loop forever:

- `XAUTOCLAIM {event} {group} {client} {max_operation_time} 0-0`

##### XREAD

- `XREADGROUP GROUP {group} {client} STREAMS {events.map ">"}`

#### Acknowledge

- `XACK {event} {group} {publish_id}`

#### Reply

- `PUBLISH {event}:{publish_id} {data}`

### IPC

Spectacles communication via IPC. Intended for local deployments where one service only needs to
communicate directly with another. Data is transmitted using JSON encoding:

```json
{
  "event": "event name",
  "data": {}
}
```

## Discord

Reference the [Discord API documentation](https://discordapp.com/developers/docs/topics/gateway#payloads) for information about packets received and sent. Packets must be JSON encoded.

### Receiving data

Only the `d` property of data received from the Discord gateway is sent to the message broker. Only OP code 0 data is sent, as the other OP codes are for internal WebSocket management.

Ex:

```
'MESSAGE_CREATE' => {
  id: 'some id',
  content: 'spectacles is awesome',
  ...
}
```

### Sending data

The full packet to send to the Discord gateway should be sent to the message broker. The shard ID is the queue name to publish to. Gateways must listen to events that correspond with shard IDs they own.

Ex:

```
'3' => {
  op: 4,
  d: {
    guild_id: 'some id',
    ...
  }
}
```

If only the guild ID is known, packets can be published as a `SEND` event. The gateway which receives this packet is then reponsible for calculating the correct shard ID and re-publishing it in the above format. This format is available so that workers do not need to be aware of the shard count (required when calculating shard ID from guild ID).

```
'SEND' => {
  guild_id: 'some id',
  packet: {
    op: 4,
    d: {
      guild_id: 'some id',
      ...
    }
  }
}
```
