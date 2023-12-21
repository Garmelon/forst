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

A message id is a string matching `e[0-9a-f]{16}`. In other words, it's a 128
bit hexadecimal integer prefixed by `e`.

Sorting events by their id puts them in chronological order.

### MessageId

A message id is a string matching `m[0-9a-f]{16}`. In other words, it's a 128
bit hexadecimal integer prefixed by `m`.

Sorting messages by their id puts them in chronological order.

### SessionId

A session id is a string matching `s[0-9a-f]{16}`. In other words, it's a 128
bit hexadecimal integer prefixed by `s`.

### UserId

A user id is a string matching `[a-z0-9-]+`. It is usually four characters long
for accounts and eight characters plus a dash for other sessions, though this is
not required. A user id uniquely identifies a user.

## Auth phase commands

### `c-auth-cookie`

Authenticate via the cookies exchanged during the HTTP handshake portion of the
WebSocket connection. This requires the client to store and present the cookies
on subsequent connections.

### `c-auth-session-id`

Authenticate via session id. This requires the client to store the session id
and present it on subsequent connections via `auth-session-id`.

### `c-auth-anon`

Don't authenticate and obtain a unique user id for this session.

## Roam phase events

### `e-goodbye`

When the server decides to close the connection, it may send a goodbye event
explaining the decision beforehand, though this is not required.

Possible reasons:

- protocol error (e.g. using roam commands during the auth phase)
- spam (the client is sending too many commands)
- login (the client has logged into an account on another connection for the same session)
- logout (the client has logged out of an account on another connection for the same session)

## Roam phase commands

### `c-enter`

Enter a room. This is required for a client to receive room events and send room
commands. A user that has entered a room in at least one client will be displayed
in the nick list.

As part of the reply to this command, the server will include the client's
current room nick.

### `c-exit`

Exit a room. After exiting a room, the server will no longer relay events for
that room, and the client can no longer issue commands for that room.

### `c-login`

## Roam phase room events

### `e-enter`

A user has entered the room.

### `e-exit`

A user has exited the room.

### `e-nick`

A user has changed their nick.

### `e-send`

A new message has been sent.

### `e-edit`

An existing message has been edited.

### `e-delete`

An existing message has been deleted.

## Roam phase room commands

### `c-nick`

Change your own nick.

### `c-send`

Send a new message.

### `c-edit`

Edit a message.

### `c-delete`

Delete a message.

### `c-get-msg`

Request a specific message.

This may be useful for clients like bots that don't want to store the entire
message history.

### `c-get-thread`

Request one or more threads, optionally starting before a specific message id.

If no message id is specified, the latest threads are returned. If a message id
is specified, the first few threads older than that id are returned.

### `c-get-log`

Request event log, optionally starting before a specific event id.

If no event id is specified, the latest events are returned. If an event id is
specified, the first few events older than that id are returned.
