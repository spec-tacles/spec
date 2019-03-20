# Spectacles cache

Spectacles uses GraphQL to describe cached data structures. This gives users considerable control over the data that the system stores as well as a descriptive API for querying it. This specification should not be used for REST objects.

## Client specifications

The following type names must be used to describe data structures of that type, but none of them are required to be present for a valid cache implementation. Each type:

- should implement properties that match those in the corresponding Discord data structure; omitted properties will not be cached
- must accept a nullable argument of `ID` type.

Mutations must not be described; the cache should ingest directly from a message broker or gateway.

- Channel
	- Message
- Guild
	- Emoji
	- GuildMember
		- VoiceState
		- Presence
	- Role
- User

The root query may implement any of the top level types; other types will be ignored. For example:

```gql
type Query {
	user(id: ID): User
}
```

Clients can query a compliant server on its root path.

## Server implementation

The server must parse the client's GraphQL schema and determine which properties of incoming objects to cache. It must then ingest these properties from a message broker and store the data in a backend. The server must serve a GraphQL endpoint on the root path that exactly matches the client's GraphQL schema.

Recommended caching backends:

- Redis
