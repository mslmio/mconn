# Mconn

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
