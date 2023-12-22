# The Forst API

A client communicates with a Forst server using JSON packets over one or more
WebSocket connections. While a single connection should usually suffice, some
clients may prefer multiple connections (e.g. multiple browser tabs).

<!-- Implementing a client should be easy, especially implementing a bot. -->
<!-- The API must be usable in the context of a web browser. -->
<!-- The API should be robust if the connection is unstable (e.g. mobile data). -->

## The connection lifecycle

A connection has two phases: The **auth phase** and the **roam phase**. In each
phase, only commands from the respective phase are allowed.

A connection starts out in the **auth phase**. Here, the client and server must
negotiate a client identity. When the client receives its identity, the
connection switches to the roam phase.

The connection now stays in the **roam phase** until it is closed. During this
phase, the client can enter and exit rooms, as well as interact with other
clients in entered rooms. The server will also send status updates (events) for
entered rooms.

## Packets

The server and client communicate using packets. A packet is a textual WebSocket
message containing a JSON object.

There are three types of packets: **Events**, **commands**, and **replies**.

- An **event** is sent from the server to the client whenever something happens
  that the client should know about.
- A **command** is sent from the client to the server. It instructs the server
  to perform an action on behalf of the client.
- The server must respond to a command with a **reply**. It describes the result
  of executing the command.

Every packet has a `type` field that describes the type of the packet. It must
have one of the following values:

- `"event"` if the packet is an **event**
- `"command"` if the packet is a **command**
- `"reply"` if the packet is a **reply**

### Events

The `data` field contains the event's payload. Its contents depend on the `name`
value of the event.

```ts
type EventPacket = {
  type: "event";
  name: string;
  data: object;
};
```

### Commands

The `data` field contains the command's payload. Its contents depend on the
`name` value of the command.

The client can provide an optional `id` value that the server has to echo in its
reply. This can be used to associate commands with their replies.

```ts
type CommandPacket = {
  type: "command";
  name: string;
  id?: string;
  data: object;
};
```

### Replies

The `result` field indicates whether the command was executed successfully or
not. If it has the value `"success"`, then the server _must_ provide the `data`
field and _must not_ provide the `error` field. If it has the value `"error"`,
then the server _must not_ provide the `data` field and _must_ provide the
`error` field.

The `data` field, if present, contains the reply's payload. Its contents depend
on the `name` value of the reply.

If the client provided an `id` value, the reply must include the exact same id.
If the client omitted the id, the server must omit it as well.

```ts
type ReplyPacket = {
  type: "reply";
  name: string;
  id?: string;
  result: "success" | "error";
  data?: object;
  error?: Error;
};
```

## Primitive types

### EventId

A message id is a string matching the regex `e[0-9A-F]{16}`. It encodes 64 bit
unsigned integer.

Event ids are unique across rooms. Sorting events by their id puts them in
chronological order.

### MessageId

A message id is a string matching the regex `m[0-9A-F]{16}`. It encodes a 64 bit
unsigned integer.

Message ids are unique across rooms. Sorting messages by their id puts them in
chronological order.

### SessionId

A session id is a string matching the regex `s[0-9A-F]{16}`. It encodes a 64 bit
unsigned integer.

### UserId

A user id is a string matching the regex `u[0-9A-F]{16}`. It encodes a 64 bit
unsigned integer.

### Account

```ts
type Account = {
  email: string;
  displayName: string;
  pingName: string;
};
```

### User

A participant in a room.

```ts
type User = {
  id: UserId;
  displayName: string;
  localDisplayName?: string;
  pingName?: string;
  account?: true;
  bot?: true;
  host?: bool;
  admin?: bool;
};
```

### Message

```ts
type Message = {
  id: MessageId;
  author: User;
  content: string;
  pinged: User[];
};
```

### RoomEvent

```ts
type RoomEvent =
  | { type: "enter"; id: EventId; user: User }
  | { type: "exit"; id: EventId; user: User }
  | { type: "user"; id: EventId; user: User }
  | { type: "send"; id: EventId; message: Message }
  | { type: "edit"; id: EventId; message: Message; by: User }
  | { type: "delete"; id: EventId; message: Message; by: User };
```

### Error

Error types include but are not limited to:

- `bad-name`:
  The server does not accept the display or ping name for the provided reason.
- `bad-content`:
  The server does not accept the message content for the provided reason.
- `bad-message`:
  The user is referencing a message that does not exist.
- `insufficient-permissions`:
  The user has insufficient permissions to execute this command.
- `not-present`:
  The user must be present in a room to execute this command.
- `password`:
  The user has tried to use an incorrect password, or has used no password where
  a password is required.

```ts
type Error =
  | { type: "bad-name"; reason: string }
  | { type: "bad-content"; reason: string }
  | { type: "bad-message"; id: MessageId }
  | { type: "insufficient-permissions" }
  | { type: "not-present" }
  | { type: "password" }
  | { type: string };
```

## Auth phase commands

These commands must only be sent during the auth phase.

### `auth-anon`

Obtain a unique user id for this session.

After the client has received a successful reply, the connection is in the roam
phase.

```ts
type Command = {};

type Reply = {
  id: UserId;
  account?: Account;
};
```

### `auth-cookie`

Authenticate via the cookies exchanged during the HTTP handshake portion of the
WebSocket connection. This requires the client to store and present the cookies
on subsequent connection handshakes.

After the client has received a successful reply, the connection is in the roam
phase.

```ts
type Command = {};

type Reply = {
  id: UserId;
  account?: Account;
};
```

### `auth-session-id`

Authenticate via session id. This requires the client to store the session id
and present it on subsequent connections via `auth-session-id`. The client must
send the last known id, if any. The server may return a different id from the id
the client sent.

After the client has received a successful reply, the connection is in the roam
phase.

```ts
type Command = {
  sessionId?: SessionId;
};

type Reply = {
  sessionId: SessionId;
  id: UserId;
  account?: Account;
};
```

## Roam phase events

These events may occur during the roam phase.

### `goodbye`

When the server decides to close the connection, it may send a goodbye event
explaining the decision beforehand, though this is not required.

```ts
type Event = {
  reason: string;
};
```

Reasons may include, but are not limited to:

- `login`:
  The client has logged into an account on a either this connection or another
  connection for the same session.
- `logout`:
  The client has logged out of an account on either this connection or another
  connection for the same session.
- `protocol`:
  The client did not follow the API specification. Example: Using roam commands
  during the auth phase.
- `spam`:
  The client sent too many commands.
- `ban`:
  The client has been banned.

## Roam phase commands

### `login`

TODO Describe `login` event

### `logout`

TODO Describe `logout` event

### `display-name`

TODO Describe `display-name` event

### `ping-name`

TODO Describe `ping-name` event

## Roam phase room events

### `enter`

A user has entered the room.

```ts
type Event = {
  id: EventId;
  room: string;
  user: User;
};
```

### `exit`

A user has exited the room.

```ts
type Event = {
  id: EventId;
  room: string;
  user: User;
};
```

### `user`

A property of a user has changed.

```ts
type Event = {
  id: EventId;
  room: string;
  user: User;
};
```

### `send`

A new message has been sent.

```ts
type Event = {
  id: EventId;
  room: string;
  message: Message;
};
```

### `edit`

An existing message has been edited.

```ts
type Event = {
  id: EventId;
  room: string;
  by: User;
  message: Message;
};
```

### `delete`

An existing message has been deleted.

```ts
type Event = {
  id: EventId;
  room: string;
  by: User;
  messageId: MessageId;
};
```

## Roam phase room commands

### `enter`

Enter a room. This is required for a client to receive room events and send room
commands. A user that has entered a room in at least one client will be present
in the user list.

TODO Room authentication, failure in reply

This command is idempotent. Entering an already entered room is allowed and has
no special effect.

Note that a client won't witness its own `enter` event.

```ts
type Command = {
  room: string;
};

type Reply = {
  present: User[];
};
```

### `exit`

Exit a room. After exiting a room, the server will no longer relay events for
that room, and the client can no longer issue commands for that room.

This command is idempotent. Exiting an already exited room is allowed and has no
special effect.

Note that a client won't witness its own `exit` event.

```ts
type Command = {
  room: string;
};

type Reply = {};
```

### `local-display-name`

Change or unset the local display name.

This command is idempotent.

Note that this will cause a `user` event if the previous local display name was
different, and may cause a `user` event even if the previous local display name
was identical.

```ts
type Command = {
  room: string;
  localDisplayName?: string;
};

type Reply = {
  user: User;
};
```

### `send`

Send a new message.

The client may optionally include a local display name. This will override the
local display name for this command only.

This command is idempotent if a `token` value is provided.

Note that this will cause a `send` event.

```ts
type Command = {
  room: string;
  localDisplayName?: string;
  token?: string;
  parent?: MessageId;
  content: string;
};

type Reply = {
  message: Message;
};
```

### `edit`

Edit a message.

The client may optionally include a local display name. This will override the
local display name for this command only.

This command is idempotent.

Note that this will cause an `edit` event.

```ts
type Command = {
  room: string;
  localDisplayName?: string;
  messageId: MessageId;
  content: string;
};

type Reply = {
  message: Message;
};
```

### `delete`

Delete a message.

The client may optionally include a local display name. This will override the
local display name for this command only.

TODO Think through how deletion interacts with the event log.

This command is idempotent. Deleting a non-existent message does not result in
an error.

Note that this will cause a `delete` event.

```ts
type Command = {
  room: string;
  localDisplayName?: string;
  messageId: MessageId;
  content: string;
};

type Reply = {};
```

### `get-users`

Request a list of users currently present in a room.

This may be useful for clients like bots that don't want to track who is present
or not.

This command is idempotent because it has no server-side effects.

```ts
type Command = {
  room: string;
};

type Reply = {
  users: User[];
};
```

### `get-message`

Request a specific message.

TODO Consider whether this should include previous edits

This may be useful for clients like bots that don't want to store the entire
message history.

This command is idempotent because it has no server-side effects.

```ts
type Command = {
  room: string;
};

type Reply = {
  message: Message;
};
```

### `get-threads`

Request one or more threads.

If no amount is specified, the server chooses an arbitrary amount. The server
may return fewer messages than the specified amount.

If a message id is specified, the youngest few threads before the id are
returned. Otherwise, the youngest few threads are returned. Returned threads are
always consecutive and ordered ascending by id (old to young).

```ts
type Command = {
  room: string;
  amount?: number;
  before?: MessageId;
};

type Reply = {
  messages: Message[];
};
```

### `get-events`

Request events from the event log.

If no amount is specified, the server chooses an arbitrary amount. The server
may return fewer messages than the specified amount.

If an event id is specified, the youngest few events before the id are returned.
Otherwise, the youngest few events are returned. Returned events are always
consecutive and ordered ascending by id (old to young).

```ts
type Command = {
  room: string;
  amount?: number;
  before?: MessageId;
};

type Reply = {
  events: RoomEvent[];
};
```
