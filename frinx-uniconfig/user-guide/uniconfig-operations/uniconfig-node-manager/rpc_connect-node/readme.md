# RPC connect-node

This RPC creates a connection to a device that is already installed.
The connection is created outside of a transaction,
so that the connection stays up even without active transactions.

!!!
Note that the transaction used in this RPC is created internally, so that no
user-created transactions are used.
!!!

The **connect-node RPC** also supports connections to stream nodes. These are
possible only if a stream node is already defined in the **install-node RPC**.
The most common use case is when a subscription stream is disconnected and a
reconnection is required.

!!!
The **connect-node RPC** only works on local Uniconfig nodes in a cluster.
!!!

## RPC parameters

- `node-id` (mandatory) - ID of a node. In case of a stream node, the node
  consists of a device node and a stream name (`<device node>_<stream name>`,
  for example `R1_NETCONF`).
- `max-connection-attempts` - Maximum number of connection attempts. The default
  value is 1.
- `between-attempts-timeout` - Timeout between connection attempts in seconds.
  The default value is 60 seconds.

## CLI Shell

The **connect-node RPC** is also included in
[UniConfig shell](https://docs.frinx.io/frinx-uniconfig/user-guide/uniconfig-operations/uniconfig-shell/).
As it takes `node-id` as input, the shell only suggests nodes that are relevant
to this RPC (nodes that are installed in UniConfig but are not yet connected).

## RPC examples

For all examples, assume that the **install RPC** request included the following
parameters:

```json
{
    "input": {
        "nodes": [
            {
                "node-id": "R1",
                "netconf": {
                    "...":  "...",
                    "stream": [
                        {
                            "stream-name": "NETCONF"
                        }
                    ]
                }
            }
        ]
    }
}
```

### Successful example - Request for a device node

```bash RPC Request
curl --location --request POST 'http://localhost:8181/rests/operations/connection-manager:connect-node' \
--header 'Authorization: Basic YWRtaW46YWRtaW4=' \
--header 'Accept: application/json' \
--header 'Content-Type: application/json' \
--header 'Cookie: Cookie_1=value' \
--data-raw '{
    "input": {
        "node-id": "R1",
        "max-connection-attempts": 1,
        "between-attempts-timeout": 1
    }
}'
```

```RPC Response, Status: 200
```

### Successful example - Request for a stream node

```bash RPC Request
curl --location --request POST 'http://localhost:8181/rests/operations/connection-manager:connect-node' \
--header 'Authorization: Basic YWRtaW46YWRtaW4=' \
--header 'Accept: application/json' \
--header 'Content-Type: application/json' \
--header 'Cookie: Cookie_1=value' \
--data-raw '{
    "input": {
        "node-id": "R1_NETCONF",
        "max-connection-attempts": 1,
        "between-attempts-timeout": 1
    }
}'
```

```RPC Response, Status: 200
```

!!!
Note that waiting for a stream node to be connected is currently not supported. This means that even if the response code is 200, the connection is not
necessarily successful as well. To test if the connection is successful or in
the process of connecting, call the RPC again (see example below) or examine the
UniConfig logs.
!!!

### Failed example - Node is already connected

```json RPC Response, Status: 409
{
    "errors": {
        "error": [
            {
                "error-tag": "in-use",
                "error-type": "application",
                "error-message": "Node R1_NETCONF is already connected"
            }
        ]
    }
}
```

### Failed example - Node is in the process of connecting

```json RPC Response, Status: 409
{
    "errors": {
        "error": [
            {
                "error-tag": "in-use",
                "error-type": "application",
                "error-message": "Stream node R1_NETCONF is in connection process."
            }
        ]
    }
}
```

### Failed example - Node does not exist

```json
{
    "errors": {
        "error": [
            {
                "error-tag": "data-missing",
                "error-type": "application",
                "error-message": "Node R1 doesn't exist in database"
            }
        ]
    }
}
```