# Spectacles spec

Details regarding how Spectacles modules communicate over various modules. Reference the [Discord API documentation](https://discordapp.com/developers/docs/topics/gateway#payloads) for information about packets received and sent. Packets should be either JSON or ETF encoded.

## AMQP

How Spectacles communicates using a message broker compatible with the AMQP protocol (likely RabbitMQ).

| Concept  | AMQP equivalent | Naming pattern                  | Description                                                                                                                                                                                          |
|----------|-----------------|---------------------------------|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| group    | exchange        | any (recommend 'gateway')       | Each Spectacles service should establish a separate exchange if it needs to send or receive messages over a broker                                                                                   |
| queue    | queue           | \<exchange\>:\[\<subgroup\>:\]\<event\> | A separate queue should be established for every exchange, event, and subgroup. A subgroup is not required, and additional queues can be added to the exchange without affecting any other services. |
| subgroup |                 | any                             | Subgroups are used to differentiate multiple queues for the same event on the same exchange. They are mostly useful for consuming an event multiple times on different services.                     |
| event    | routing key     | "t" property from the gateway   | Ensure that the gateway only publishes events that will be consumed.                                                                                                                                 |

Queues and exchanges must be declared as durable to prevent data loss in the event of multiple service failures.

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

## Caching

How Spectacles caches Discord objects using a k/v storage.

| Discord Entity | Field Naming pattern           | Key Naming pattern | Type                               | Description                                          |
|----------------|--------------------------------|--------------------|------------------------------------|------------------------------------------------------|
| User           | USERS                          | USER_ID            | Hashmap                            | Discord Users stored by ID                           |
| Guilds         | GUILDS                         | GUILD_ID           | Hashmap                            | Discord Guilds stored by ID.                         |
| Members        | MEMBERS:GUILD_ID               | MEMBER_ID          | Hashmap per Guild                  | Discord Members stored by ID.                        |
| Voice States   | VOICE_STATES:GUILD_ID          | USER_ID            | Hashmap per Guild                  | Discord Voice States stored by User ID.              |
| Presences      | PRESENCES:USER_ID              | USER_ID            | Hashmap                            | Discord Presences stored by User ID.                 |
| Channels       | CHANNELS                       | CHANNEL_ID         | Hashmap, Set per Guild storing IDs | Channels stored by Channel ID.                       |
| Roles          | ROLES                          | ROLE_ID            | Hashmap per Guild                  | Guild Roles stored by Role ID.                       |
| Emojis         | EMOJIS                         | EMOJI_ID           | Hashmap                            | Guild Emojis stored by Emoji ID.              |
| Messages       | MESSAGES:CHANNEL_ID:MESSAGE_ID | MESSAGE_ID         | Key                                | Channel Messages stored by Channel ID & Messages ID. |

Cached Guilds should not contain following keys: `VoiceStates`, `Roles`, `Emojis`, `Channels`, `Members`, `Presences`.
The Client should allow to set a TTL time for Message Objects, this can be done through an optional property which should default to 10 minutes.