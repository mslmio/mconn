# Mconn

<!-- toc -->

- [Overview](#overview)
- [Logical Message](#logical-message)
- [Message Fragment](#message-fragment)
  - [`msg_type`](#msg_type)
  - [`has_msg_id`](#has_msg_id)
  - [`has_peer_msg_id`](#has_peer_msg_id)
  - [`has_more`](#has_more)
  - [`len_of_payload_len`](#len_of_payload_len)
  - [`msg_id`](#msg_id)
  - [`peer_msg_id`](#peer_msg_id)
  - [`payload_len`](#payload_len)
  - [`payload`](#payload)
  - [Notes](#notes)
- [Message Security](#message-security)
- [Message Multiplexing](#message-multiplexing)
- [Message Ordering](#message-ordering)
- [Message Types](#message-types)
    - [Handshake Message](#handshake-message)
      - [`proto_vsn`](#proto_vsn)
      - [`encryption_scheme`](#encryption_scheme)
      - [`encryption_init_data`](#encryption_init_data)
      - [`accepted`](#accepted)
      - [Encryption Schemes](#encryption-schemes)
        - [TLS](#tls)
    - [Close Message](#close-message)
    - [Application Message](#application-message)
- [Notes](#notes-1)
    - [Request & Response](#request--response)

<!-- tocstop -->

## Overview

Mconn is a protocol layer above TCP for server-to-server network communication.

It provides a feature set that gives reliable message boundaries via framing,
multiplexing via fragmentation, security via encryption, ordering via channels,
flexibility via full-duplex communication and compactness via small and
variable-sized message headers.

Mconn is not responsible for features such as ensuring that messages were
"accepted" (by some definition of "accepted") by applications receiving
messages on the other side of a connection, and thus Mconn is also not
responsible for retrying "failed" (by some definition of "failed") messages.
Essentially, Mconn cannot tell what an application has defined as "accepted" or
"failed", as the application layer is not its responsibility. It follows that
features like timeouts and proxying are also out of scope - anything that
relies on application-level decision making or implies some particular network
architecture (as in the case of proxies) is out of scope.

However, Mconn contains optional message IDs for when such features are
required, allowing applications to have further control of the lifecycle of
Mconn messages by tracking them via their IDs.

## Logical Message

A "logical message" (or often just "message") is defined as a logical set of
data that one side of an Mconn connection would like to send to another. An
example of a logical message would be the entire contents of a given file.
Another example would be a single JSON object.

When sending such data over TCP, the network layer that Mconn builds upon, the
data is not guaranteed to arrive on the other side at exactly the same time as
one big chunk - it will likely come in smaller chunks and the receiver is
responsible for querying for the data, building up in a buffer until the entire
message is received. Even after complete reception, data boundaries must be
manually determined.

In Mconn, data is received as logical sets of data, rather than arbitrary
chunks. So the receiver querying for a message can expect that, for example,
the entire file will be received at once, or the entire JSON object.

However, streaming APIs are also available in Mconn, where users may receive
chunks of the overall message if needed. These chunks are called "message
fragments", which we cover next.

## Message Fragment

A message, when sent over the network, is sent as message fragments (or just
"fragments"), which are framed chunks of a logical message. The receiver will,
by default, receive logical messages, but via streaming APIs can receive the
individual fragments as well.

The majority of small messages will only require a single fragment. In that
case, we say that a logical message is equal to the single message fragment
that was required for sending it.

It is up to the Mconn implementation to decide what constitutes a point of
fragmentation within a logical message. This may be configurable by a user,
e.g. "only fragment on 1MB data boundaries". There should be reasonable limits
to configurability, as fragmentation is a key feature required for multiplexing
(to be discussed below), and as such making a bad configuration could adversely
affect core Mconn features. However, it is left to the implementation to decide
what level of control shall be given to its users.

Each message fragment consists of a header with metadata, which includes, for
example, bit fields that tell the size of subsequent variable-sized fields. The
tail of the fragment consists of the "payload", which is the data being sent by
the Mconn user.

```
+------------------------------------+
|              msg_type              |  3 bits ----+
+------------------------------------+             |
|             has_msg_id             |  1 bit      |
+------------------------------------+             |
|           has_peer_msg_id          |  1 bit      | 1 byte
+------------------------------------+             |
|              has_more              |  1 bit      |
+------------------------------------+             |
|         len_of_payload_len         |  2 bits ----+
+------------------------------------+
|               msg_id               |  0 or 8 bytes
+------------------------------------+
|            peer_msg_id             |  0 or 8 bytes
+------------------------------------+
|             payload_len            |  `2^len_of_payload_len` bytes
+------------------------------------+
|               payload              |  `payload_len` bytes
+------------------------------------+
```

### `msg_type`

A 3-bit field containing an enumeration value that indicates what type of
message this is.

The message type dictates how the payload is to be interpreted.

Note that all fragments of a message must have the same message type.

The following values may be assigned:

- `0`: Handshake Message.
- `1`: Close Message.
- `2-6`: reserved.
- `7`: Application Message.

The specific semantics of each message types are discussed in a section below.

### `has_msg_id`

A 1-bit field boolean which indicates whether the `msg_id` field is 0 bytes or
8 bytes long. If `has_msg_id == 0`, then `byte_length(msg_id) == 0`, otherwise
`byte_length(msg_id) == 8`.

This field must always be `1` (turned on) when multiple fragments are expected
for a message, so that the fragments can be connected together into a single
logical message reliably in the context of multiplexing (discussed later).

The use case for an optional message ID is when tracking a message's lifecycle
is not important for either application, where the only thing that matters is
speed and compactness. In such a case, the message ID can be disabled, saving
significant space in the header.

### `has_peer_msg_id`

A 1-bit field boolean which indicates whether the `peer_msg_id` field is 0
bytes or 8 bytes long. If `has_peer_msg_id == 0`, then
`byte_length(peer_msg_id) == 0`, otherwise `byte_length(peer_msg_id) == 8`.

The use case for an optional peer message ID is when trying to associate a
message being sent from one peer with a message being sent by the other, e.g.
in a request/response communication cycle.

### `has_more`

A 1-bit field boolean which indicates whether more fragments will be sent for
this message. If `has_more == 1`, then more fragments are to be expected for
the given message, otherwise this is the last fragment for the message.

This field must be set even if it is only _speculated_ that more fragments may
be needed. In such a case, when it is certain that the message is complete, a
fragment with an empty payload (i.e. `payload_len == 0`) may be sent with
`has_more == 0` to indicate to the other side that the message is complete.

For example, when a write-side stream API is provided to the user, the user may
write a part of the message, and pause for some time. During that time the
partial message could be sent as a fragment, with `has_more == 1`. But the user
may now decide that no further data is actually needed. When that decision is
given to Mconn, one last "empty" fragment will be sent with `has_more == 0` and
`payload_len == 0`, so that the other side may consider the message completed.

### `len_of_payload_len`

A 2-bit field unsigned number (0, 1, 2 or 3) which serves as the exponent `N`
in the formula `byte_length(payload_len) == 2^N`. This field is thus used to
determine how large the field `payload_len` is, and thus also the maximum
range that `payload_len` can represent.

The use case for this is small messages: many payloads may not be more than a
few bytes in size, and thus a default 8-byte `payload_len` would unnecessarily
make up a huge percentage of the overall message size. With variable-sized
`payload_len` fields, we can minimize this cost in such use cases and save
significant bandwidth over longer periods of time.

### `msg_id`

A 0- or 8-byte long, incremented-by-2 message ID. The length is discussed in
`has_msg_id`.

The connection peer which starts the connection must always start with
`msg_id == 1`, and increment by 2 for selecting the next message ID to use. The
connection peer that received the connection request must always start with
`msg_id == 2`, and increment by 2 for selecting the next message ID to use.
Effectively, the connection starter will always have even-numbered message IDs,
and the other side will have odd-numbered message IDs. A message ID of 0 may be
used internally in implementations to represent "no ID". All fragments of a
message must have the same message ID.

This dichotomy exists to allow a trivial form of stateless message ID
synchronization. Because each side's set of possible message IDs are fixed and
completely disjoint, no communication is needed to prevent message ID
conflicts. Because no conflicts exist, each side can generate new messages
containing message IDs knowing that the message will be globally unique in the
context of the connection.

Note that this dichotomy only reduces the ID space by a factor of 2, i.e.
`2^63 - 1` unique message IDs are still possible from each side, each trivially
generatable by simple addition.

On an extremely long-lived connection with lots of messages being exchanged,
one may fear that the ID space will be exhausted. We present an example to
lighten such burden: with `has_msg_id == 1` and
`byte_length(payload_len) == 1` and a 1-byte payload, 11 bytes are needed to
represent the message, so sending `2^63 - 1` such messages would amount to
sending ~88 exabytes of data (i.e. ~90,112 petabytes). If this is still a
problem, simply restart the connection.

### `peer_msg_id`

A 0- or 8-byte long message ID, whose values must be within the set of message
IDs available to the receiving peer.

A peer message ID of 0 may be used internally in implementations to represent
"no ID". All fragments of a message must have the same peer message ID.

### `payload_len`

A 1-, 2-, 4- or 8-byte length unsigned number representing the number of bytes
in the payload. The length is determined by `len_of_payload_len`.

There are no minimum value requirements relative to the length in bytes of the
field; it is perfectly allowed to have a 8-byte field with a stored value of
`0`, indicating an empty payload.

### `payload`

A `payload_len` byte field which contains arbitrary data.

The interpretation of this data depends upon the message type, but even within
each message type there may be further delegation of interpretation up to higher
levels, such as the application in the case of an "application" message type.

### Notes

`has_msg_id` and `has_more`, together, only have 3 possible configurations:

- `has_msg_id == 0 && has_more == 0`
- `has_msg_id == 1 && has_more == 0`
- `has_msg_id == 1 && has_more == 1`

Namely, it is not possible that `has_msg_id == 0 && has_more == 1`, since if
multiple fragments exist (i.e. `has_more == 1`) for a single message, then it's
impossible for the message to not have an ID (i.e. `has_msg_id == 0`),
otherwise the fragments could not be connected into a single message reliably.

As an interesting thought, if only 2 instead of 3 valid states were possible,
then only 1 bit would be needed to represent the combination of `has_msg_id`
and `has_more`. But since 3 states are allowed, we require 2 bits, and separate
each variable into a separate bit for ease.

## Message Security

Encrypting messages is required by Mconn, and the encryption algorithm must be
a strong symmetric-key block cipher, which is chosen during negotiation in the
Mconn handshake (discussed later).

When an algorithm is chosen, a fragment is encrypted as the last step before it
is sent over the wire. The entirety of the fragment is encrypted, including the
header and payload, and decryption occurs on the encrypted fragment. Decryption
is not meant to occur on the concatenation of encrypted fragments.

Further details of the cryptographic process is left to the specific algorithm
chosen during the Mconn handshake.

## Message Multiplexing

Messages may be multiplexed with other messages via a scheduling algorithm
which sends fragments of different messages in an order not necessarily
dependent upon the order in which messages were provided to Mconn.

For example, a message with a 1GB-sized payload can be given to Mconn first,
which can then be fragmented into a size chosen by the implementation, and sent
in those chunks one at a time. Before the fragments of the initial message are
all completely sent, another message with a 1KB-sized payload may be given, and
the implementation's fragment scheduling algorithm may decide that this message
can be sent as a single fragment and can be sent right now. Thus, the entirety
of the second message would be received by the other side of the connection
before the first message, even though it was input later than the first.

The use case of multiplexing is to prevent head-of-line blocking, where a very
large message which may take a long time to fully deliver will block all
messages input before it, no matter how small those later messages are. It also
allows for more "important" messages to arrive on the other side before less
important ones. For example, a large file may be getting "downloaded" by the
other peer, and during this process a new message is input that has high urgency
and is not very large; the urgent message's fragment(s) can be fast-tracked to
the front of the queue by the fragment scheduling algorithm and sent
immediately, without having to cancel the download.

Note that even with multiplexing, Mconn must still guarantee that fragments of
a message must all arrive in the correct order. In other words, although
the ordering of messages is not guaranteed, the ordering of fragments within a
message must be guaranteed. For example, if a message has a 3MB payload, and
the implementation decides to send it as 3 fragments each containing 1MB of
that payload, where fragment `A` contains the first MB, fragment `B` contains
the second MB, and fragment `C` contains the third MB, then the implementation
must guarantee that fragments `A`, `B` and `C` are sent in that order for the
given message, even if fragments of other messages are sent interspersed within
this order.

## Message Ordering

Since multiplexing implies that the order in which messages arrive is not
guaranteed, a "channel" can be constructed within one side of an Mconn
connection which guarantees that all messages sent via that channel are
ordered. Essentially, the implementation's fragment scheduling algorithm would
guarantee that all messages within a channel are sent to the other side in the
same order in which they were input, and also guarantees that the fragments of
these messages are never multiplexed amongst each other. In essence, all
messages within a channel look as if they were a part of one big message from
the perspective of the scheduling implementation.

By default, all messages are effectively within their own one-message channel,
and thus only have fragment-level ordering guarantees but not message-level
ordering guarantees, and are multiplexed as per the implementation's fragment
scheduling algorithm.

Note that message ordering via channels is strictly an implementation
construct; information about a channel is not included nor needed in the wire
protocol.

## Message Types

Different message types exist to allow correct interpretation of message
payloads.

All except one message type are specific to the Mconn protocol and its internal
functioning. The application message type is meant to store arbitrary data that
applications would use in layers above Mconn.

#### Handshake Message

A handshake message is meant to specify, agree upon, and initialize certain
persistent connection parameters.

A handshake message request is the first message to be sent when a TCP
connection successfully completes, and is sent by the connection initiating
peer. A handshake message response is sent after the receiver processes the
request and confirms whether it can support the requested connection
parameters. Handshake messages must not be sent after the initial handshake is
completed, otherwise the receiving peer may either ignore it, log it, and/or
close the connection.

The protocol version determines how to interpret data after all of the protocol
version byte. The following specification only contains the format of version
0.

The following shows the order of messages to be sent for this version of the
protocol:

1. Handshake message request.
2. Handshake message response.
3. Encryption scheme messages (if any).

##### Handshake Request

The following shows the format of the handshake request:

```
+------------------------------------+
|              proto_vsn             |  1 byte
+------------------------------------+
|          encryption_scheme         |  1 byte
+------------------------------------+
|         encryption_init_data       |  variable-length
+------------------------------------+
```

The meaning of the fields are discussed below.

##### `proto_vsn`

The version of the Mconn protocol that is being used.

Specifically, this controls the interpretation of all data coming after this
field itself.

##### `encryption_scheme`

The encryption scheme is used to determine how encryption will occur on this
connection.

Specifically, it allows for specifying how to initiate the encryption protocol,
and decides the interpretation of `encryption_init_data`.

The following values are allowed in `encryption_scheme`:

- `0`: no encryption.
- `1-19`: reserved.
- `20`: TLS, where handshake request sender is the client.
- `22-255`: reserved.

The details of each encryption scheme is discussed below.

##### `encryption_init_data`

This field is variable-length, and depends wholly on the value specified in
`encryption_scheme`; it may thus be 0 or more bytes.

##### Handshake Response

The following shows the format of the handshake response:

```
+------------------------------------+
|              accepted              |  1 byte
+------------------------------------+
```

##### `accepted`

Whether the connection parameters specified in the request are accepted by the
receiving peer.

If `accepted == 0`, the request is rejected and the connection will be closed
at a TCP level immediately. Otherwise the request is accepted and any further
handshake message exchanges required to complete the handshake shall commence.

##### Encryption Schemes

We now cover the different encryption schemes that may be used and how they
are finalized after a affirmative handshake message response is given.

Note that after an encryption scheme is finalized, all further messages *must*
be encrypted and decrypted using the agreed-upon scheme. The initial handshake
response message which accepts or rejects the request *must not* be encrypted,
however.

###### TLS

This scheme indicates that immediately after an affirmative handshake response
is received, the TLS handshake will occur. No further messages, after the full
TLS handshake is complete, are needed to finalize this encryption scheme.

#### Close Message

A close message tells the other side that the sending side desires to close the
connection.

A close message request can be sent by either side. The receiver processes the
request, performs any necessary cleanup, and sends the response.

##### Close Request

The following shows the format of the close request:

```
+------------------------------------+
|             close_mode             |  1 byte
+------------------------------------+
```

##### `close_mode`

After a close request has been sent, the sender expects a close response within
a specified timeout. If a response is not sent after a timeout is initiated,
abrupt TCP connection close will occur. The timeout is started depending on the
close mode as follows:

- immediately send the response, or else an abrupt TCP connection close will
  occur in a timeout. Any pending messages are discarded and any new messages
  are ignored.
- any messages whose fragments have been received so far will continue to
  be accepted until they are complete, after which a close response is
  expected with a timeout. New messages are ignored.
- all messages whose fragments have been received so far, as well as any
  messages in the sending queue of the other side, will be accepted, after
  which a close response is expected with a timeout.

In all three close modes, each side immediately rejects any new messages from the user.

##### Close Response

The close response message has no payload. Instead, the ID of the request
message is set as the peer ID for the response. The side that sent the close
request then looks for a close message whose peer message ID is equal to the
request message ID.

#### Application Message

Application messages contain payloads with meaning beyond the scope of Mconn.
It is up to the user of Mconn to decide what to do with such payloads.

## Notes

Mconn is very flexible and powerful, with a large variety of features within a
single network layer. It can support lots of different application-layer
features quite easily with a bit of extra work.

It is therefore suggested for implementations to provide basic support for
common application-layer use cases. Some specific suggestions are provided
below.

#### Request & Response

Request & response protocols can be supported by using message IDs & peer
message IDs to keep track of message life-cycles.

Specifically, a "request message" (or just "request") may be generated by
creating and sending a message that forcefully has a message ID, and then
awaiting a "response message" (or just "response"). A response may be generated
by the peer by also creating and sending a message that forcefully has a
message ID, but which also assigns the peer message ID for that message to be
equal to the request message ID.

The receiving side will then perform a message read, specifying to the
implementation that it is specifically looking for a message whose peer message
ID is equal to the request message ID.
