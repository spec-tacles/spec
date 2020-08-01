- [Ratelimits](https://github.com/spec-tacles/spec/blob/master/ratelimits.md)

# Spectacles spec

Details regarding how Spectacles modules communicate over various modules. Reference the [Discord API documentation](https://discordapp.com/developers/docs/topics/gateway#payloads) for information about packets received and sent. Packets must be JSON encoded.

## Receiving data

Only the `d` property of data received from the Discord gateway is sent to the message broker. Only OP code 0 data is sent, as the other OP codes are for internal WebSocket management.

Ex:

```
'MESSAGE_CREATE' => {
  id: 'some id',
  content: 'spectacles is awesome',
  ...
}
```

## Sending data

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

## Protocols

Supported communication protocols.

### AMQP

How Spectacles communicates using a message broker compatible with the AMQP protocol (likely RabbitMQ).

| Concept  | AMQP equivalent | Naming pattern                  | Description                                                                                                                                                                                          |
|----------|-----------------|---------------------------------|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| group    | exchange        | any (recommend 'gateway')       | Each Spectacles service should establish a separate exchange if it needs to send or receive messages over a broker                                                                                   |
| queue    | queue           | \<exchange\>:\[\<subgroup\>:\]\<event\> | A separate queue should be established for every exchange, event, and subgroup. A subgroup is not required, and additional queues can be added to the exchange without affecting any other services. |
| subgroup |                 | any                             | Subgroups are used to differentiate multiple queues for the same event on the same exchange. They are mostly useful for consuming an event multiple times on different services.                     |
| event    | routing key     | "t" property from the gateway   | Ensure that the gateway only publishes events that will be consumed.                                                                                                                                 |

Queues and exchanges must be declared as durable to prevent data loss in the event of multiple service failures.

### IPC

Spectacles communication via IPC. Intended for local deployments where one service only needs to communicate directly with another. Data is transmitted using JSON encoding:

```json
{
  "event": "event name",
  "data": {}
}
```
