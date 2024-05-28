# RPC check-nodes-connection

This RPC checks if specified devices are reachable (health-check). This is done by session creation, 
single request execution (most lightweight one), session cleanup.

RPC input specifies target-nodes on which UniConfig checks connection. Each target-node is executed in parallel and 
whole RPC is outside the uniconfig transaction concept. Output describes only failed health-check operations. 
Input may also contain `connection-timeout` which is overall timeout for operation -> establishing session amd single request to device. 
This timeout is applied individually per each target-node (default value is 5 seconds).


## RPC examples

### Successful example

RPC input specify a device R1 without specifying connection-timeout parameter.

```bash RPC Request
curl --location --request POST 'http://localhost:8181/rests/operations/connection-manager:check-nodes-connection' \
--header 'Authorization: Basic YWRtaW46YWRtaW4=' \
--header 'Content-Type: application/json' \
--data-raw '{
    "input": {
        "target-nodes" : {
            "node" : ["R1"]
        }
    }
}'
```

```json RPC Response, Status: 204
```

### Failed example

RPC input specifies device R1 (reachable), R2 (not reachable). 
Input also contains `connection-timeout` parameter set for 3 seconds. 
Operation was successful only for device R1. Output describes output for failed device R2.

```bash RPC Request
curl --location --request POST 'http://localhost:8181/rests/operations/connection-manager:check-nodes-connection' \
--header 'Authorization: Basic YWRtaW46YWRtaW4=' \
--header 'Content-Type: application/json' \
--data-raw '{
    "input": {
        "target-nodes" : {
            "node" : ["R1", "R2"]
        },
        "connection-timeout": 3
    }
}'
```

```json RPC Response, Status: 500
{
    "errors": {
        "error": [
            {
                "error-tag": "operation-failed",
                "error-info": {
                    "node-id": "R2",
                    "error": "Health-check operation on node R2 timed out after 3 seconds",
                    "topology-id": "gnmi-topology"
                },
                "error-message": "Could not invoke health-check operation for node 'R2' and topology 'gnmi-topology'",
                "error-type": "application"
            }
        ]
    }
}
```

### Failed example

RPC input specifies device R1 which is not installed.

```bash RPC Request
curl --location --request POST 'http://localhost:8181/rests/operations/connection-manager:check-nodes-connection' \
--header 'Accept: application/json' \
--header 'Content-Type: application/json' \
--data-raw '{
    "input": {
        "target-nodes" : {
            "node" : ["R1"]
        }
    }
}'
```

```json RPC Response, Status: 404
{
    "errors": {
        "error": [
            {
                "error-tag": "data-missing",
                "error-info": {
                    "node-id": "R1"
                },
                "error-message": "Node with ID 'R1' is not installed",
                "error-type": "application"
            }
        ]
    }
}
```
