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

Every packet has a `name` field and a `data` field. The `name` field contains
the name of the event or command. The `data` field contains a json object that
is the payload of the event, command, or reply.

**Command** and **reply** packets have an optional field `id` of type string
that can be used by the client to associate replies with commands. When the
client sends a command, it may include an arbitrary id. In its reply to the
command, the server must include the exact same id. If the client omitted the
id, the server must omit it as well.

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
  | {
      type: "enter";
      id: EventId;
      user: User;
    }
  | {
      type: "exit";
      id: EventId;
      user: User;
    }
  | {
      type: "user";
      id: EventId;
      user: User;
    }
  | {
      type: "send";
      id: EventId;
      message: Message;
    }
  | {
      type: "edit";
      id: EventId;
      by: User;
      message: Message;
    }
  | {
      type: "delete";
      id: EventId;
      by: User;
      message: Message;
    };
```

## Auth phase commands

These commands must only be sent during the auth phase.

### `auth-anon`

Obtain a unique user id for this session.

After the client has received the reply, the connection is in the roam phase.

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

After the client has received the reply, the connection is in the roam phase.

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

After the client has received the reply, the connection is in the roam phase.

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

- `protocol`: The client did not follow the API specification. Example: Using
  roam commands during the auth phase.
- `spam`: The client sent too many commands.
- `login`: The client has logged into an account on either this connection or
  another connection for the same session.
- `logout`: The client has logged out of an account on either this connection or
  another connection for the same session.

## Roam phase commands

### `login`

TODO Describe `login` event

### `logout`

TODO Describe `logout` event

### `display-name`

TODO Describe `display-name` event

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

The server may reject the command because the user is not in the specified room.
If so, it will return the result `not-present`.

The server may reject the command for an arbitrary reason. If so, it will return
the result `rejected` and provide a human-readable reason in its reply.

This command is idempotent.

Note that this will cause a `user` event if the previous local display name was
different, and may cause a `user` event even if the previous local display name
was identical.

```ts
type Command = {
  room: string;
  localDisplayName?: string;
};

type Reply =
  | {
      result: "success";
      user: User;
    }
  | { result: "not-present" }
  | {
      result: "rejected";
      reason: string;
    };
```

### `send`

Send a new message.

The server may reject the command because the user is not in the specified room.
If so, it will return the result `not-present`.

The server may reject the command because the parent does not exist. If so, it
will return the result `nonexistent-parent`.

The server may reject the command for an arbitrary reason. If so, it will return
the result `rejected` and provide a human-readable reason in its reply.

This command is idempotent if a `token` value is provided.

Note that this will cause a `send` event.

```ts
type Command = {
  room: string;
  token?: string;
  parent?: MessageId;
  content: string;
};

type Reply =
  | {
      result: "success";
      message: Message;
    }
  | { result: "not-present" }
  | { result: "nonexistent-parent" }
  | {
      result: "rejected";
      reason: string;
    };
```

### `edit`

Edit a message.

The server may reject the command because the user is not in the specified room.
If so, it will return the result `not-present`.

The server may reject the command because the user has insufficient permissions
to edit the message. If so, it will return the result
`insufficient-permissions`.

The server may reject the command because the message does not exist. If so, it
will return the result `nonexistent`.

The server may reject the command for an arbitrary reason. If so, it will return
the result `rejected` and provide a human-readable reason in its reply.

This command is idempotent.

Note that this will cause an `edit` event.

```ts
type Command = {
  room: string;
  messageId: MessageId;
  content: string;
};

type Reply =
  | {
      result: "success";
      message: Message;
    }
  | { result: "not-present" }
  | { result: "insufficient-permissions" }
  | { result: "nonexistent" }
  | {
      result: "rejected";
      reason: string;
    };
```

### `delete`

Delete a message.

TODO Think through how deletion interacts with the event log.

The server may reject the command because the user is not in the specified room.
If so, it will return the result `not-present`.

The server may reject the command because the user has insufficient permissions
to delete the message. If so, it will return the result
`insufficient-permissions`.

This command is idempotent. Deleting a non-existent message does not result in
an error.

Note that this will cause a `delete` event.

```ts
type Command = {
  room: string;
  messageId: MessageId;
  content: string;
};

type Reply = {
  result: "success" | "not-present" | "insufficient-permissions";
};
```

### `get-users`

Request a list of users currently present in a room.

This may be useful for clients like bots that don't want to track who is present
or not.

The server may reject the command because the user is not in the specified room.
If so, it will return the result `not-present`.

This command is idempotent because it has no server-side effects.

```ts
type Command = {
  room: string;
};

type Reply =
  | {
      result: "success";
      users: User[];
    }
  | { result: "not-present" };
```

### `get-message`

Request a specific message.

TODO Consider whether this should include previous edits

This may be useful for clients like bots that don't want to store the entire
message history.

The server may reject the command because the user is not in the specified room.
If so, it will return the result `not-present`.

The server may reject the command because the message does not exist. If so, it
will return the result `nonexistent`.

This command is idempotent because it has no server-side effects.

```ts
type Command = {
  room: string;
};

type Reply =
  | {
      result: "success";
      message: Message;
    }
  | { result: "not-present" }
  | { result: "nonexistent" };
```

### `get-threads`

Request one or more threads.

If no amount is specified, the server chooses an arbitrary amount. The server
may return fewer messages than the specified amount.

If a message id is specified, the youngest few threads before the id are
returned. Otherwise, the youngest few threads are returned. Returned threads are
always consecutive and ordered ascending by id (old to young).

The server may reject the command because the user is not in the specified room.
If so, it will return the result `not-present`.

```ts
type Command = {
  room: string;
  amount?: number;
  before?: MessageId;
};

type Reply =
  | {
      result: "success";
      messages: Message[];
    }
  | { result: "not-present" };
```

### `get-events`

Request events from the event log.

If no amount is specified, the server chooses an arbitrary amount. The server
may return fewer messages than the specified amount.

If an event id is specified, the youngest few events before the id are returned.
Otherwise, the youngest few events are returned. Returned events are always
consecutive and ordered ascending by id (old to young).

The server may reject the command because the user is not in the specified room.
If so, it will return the result `not-present`.

```ts
type Command = {
  room: string;
  amount?: number;
  before?: MessageId;
};

type Reply =
  | {
      result: "success";
      events: RoomEvent[];
    }
  | { result: "not-present" };
```
