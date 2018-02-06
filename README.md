# Spectacles spec

Details regarding how Spectacles modules communicate over various modules. Reference the [Discord API documentation](https://discordapp.com/developers/docs/topics/gateway#payloads) for information about packets received and sent.

## AMQP

How Spectacles communicates using a message broker compatible with the AMQP protocol (likely RabbitMQ). Sending and receiving data is done over separate exchanges of arbitrary names. The event name is used as a routing key, and by default the client should subscribe to and pull from from queues named in the pattern of `[exchange name]:[event name]`.

### Receiving data

Only the `d` property of data received from the Discord gateway is sent to the message broker. Only OP code 0 data is sent, as the other OP codes are for internal WebSocket management. The `t` property of received data determines the queue name to publish to.

Ex:

```
'MESSAGE_CREATE' => {
  id: 'some id',
  content: 'spectacles is awesome',
  ...
}
```

### Sending data

The full packet to send to the Discord gateway should be sent to the message broker. The shard ID is the queue name to publish to.

Ex:

```
'4' => {
  op: 4,
  d: {
    guild_id: 'some id',
    ...
  }
}
