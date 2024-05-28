# RPC commit

The external application stores the intended configuration under nodes in the
UniConfig topology. The trigger for executing the configuration is an RPC
commit. RPC output describes the result of the commit.

The configuration of nodes consists of the following phases:

1. **Lock and validate configured nodes**: Lock all modified nodes using
   PostgreSQL advisory locks and validate fingerprints. If another transaction
   attempts to commit overlapping nodes or another transaction has already
   changed one of the nodes, commit will fail at this step.
2. **Write configuration to device**: Push calculated changes onto the device
   without committing these changes.
3. **Validate configuration**: Validate the written configuration from the view
   of constraints and consistency. This phase can be skipped with the
   `do-validate` flag.
4. **Confirmed commit**: Lock the device configuration so that no other
   transaction can touch the device. This phase can be skipped with the
   `do-confirmed-commit` flag.
5. **Confirming commit (submit configuration)**: Persist all changes on devices
   and in the PostgreSQL database. The UniConfig transaction is closed.
6. **Rollback**: Restore the configuration to its previous state if the
   configuration process fails. When configuring multiple devices in a single
   transaction and the process fails on one particular device, the rollback
   procedure is applied to **all touched devices**. This is done with the auto
   rollback procedure, which is by enabled by default. The procedure can be
   disabled with the `do-rollback` flag in the RPC input, in which case **only
   failed devices are rolled back**. 

!!!
The `skip-unreachable-nodes` flag controls whether unreachable nodes are
skipped when the RPC commit is sent. When set to `true`, nodes that are not
reachable are skipped and others are configured. The default value is `false`.
!!!

![RPC commit](RPC_commit-RPC_commit.svg)

Commit invokes all nodes touched in the transaction. There are no target nodes
in the RPC input. Steps 3 and 4 only apply to nodes that support these operations.

If a node fails
for any reason, the entire RPC fails. After the commit RPC, the UniConfig
transaction is closed regardless of the result.

!!!
If one of the nodes uses a confirmed commit (step 4) that does not fail, it is
necessary to issue the submitted configuration (step 5) within the timeout
period. Otherwise the node configuration issued by the confirmed commit is
reverted to its state before the confirmed commit (i.e., confirmed commit makes
only temporary configuration changes). The default timeout period is 600 seconds
(10 minutes), which can be changed in the installation request.
!!!

The following diagrams describe all five commit steps in more detail:

**1. Lock and validate configured nodes**

![Locking nodes](RPC_commit-locking-phase-Locking_phase.svg)

**2. Write configuration to device**

![Configuration phase](RPC_commit-configuration-phase-Configuration_phase.svg)

**3. Validate configuration**

![Validation phase](RPC_commit-validation-phase-Validation_phase.svg)

**4. Confirmed commit**

![Confirmed commit](RPC_commit-confirmed-commit-phase-Confirmed_commit_phase.svg)

**5. Confirming commit (submit configuration)**

![Confirming commit](RPC_commit-confirming-commit-Confirming_commit_phase.svg)

The next diagram shows the rollback procedure that must be executed after a
failed commit on nodes that have already been configured and do not support the
`candidate` datastore.

![Rollback operation](RPC_commit_rollback-Rollback_changes.svg)

## RPC examples

### Successful example

UniConfig commits nodes R1 and R2 that have been changed in the actual
transaction.

```bash RPC Request
curl --location --request POST 'http://localhost:8181/rests/operations/uniconfig-manager:commit' \
--header 'Accept: application/json' \
--header 'Content-Type: application/json' \
--data-raw '{
    "input": {
    }
}'
```

```RPC Response, Status: 204
```

### Successful example

Nodes R1 and R2 have been changed. RPC input includes the flag to disable the
confirmed-commit phase. UniConfig commits all touched nodes.

```bash RPC Request
curl --location --request POST 'http://localhost:8181/rests/operations/uniconfig-manager:commit' \
--header 'Accept: application/json' \
--header 'Content-Type: application/json' \
--data-raw '{
    "input": {
        "do-confirmed-commit": false,
    }
}'
```

```RPC Response, Status: 204
```

### Successful example

If there are no touched nodes, the request finishes successfully.

```bash RPC Request
curl --location --request POST 'http://localhost:8181/rests/operations/uniconfig-manager:commit' \
--header 'Accept: application/json' \
--header 'Content-Type: application/json' \
--data-raw '{
    "input": {
    }
}'
```

```RPC Response, Status: 204
```

### Failed example

Node R1 has failed because it failed the validation step.

```bash RPC Request
curl --location --request POST 'http://localhost:8181/rests/operations/uniconfig-manager:commit' \
--header 'Accept: application/json' \
--header 'Content-Type: application/json' \
--data-raw '{
    "input": {
    }
}'
```

```json RPC Response, Status: 500
{
  "errors": {
    "error": [
      {
        "error-type": "application",
        "error-tag": "validation-failed",
        "error-message": "RemoteDevice{R2}: Validate failed. illegal reference /orgs/org[name='TESTING-PROVIDER']/traffic-identification/using-networks",
        "error-info": {
          "node-id": "R2",
          "configuration-status": "fail",
          "rollback-status": "complete"
        }
      }
    ]
  }
}
```

### Failed example

Node R1 has failed because the confirmed commit failed. The validation step is
skipped because of the `do-validate` flag.

```bash RPC Request
curl --location --request POST 'http://localhost:8181/rests/operations/uniconfig-manager:commit' \
--header 'Accept: application/json' \
--header 'Content-Type: application/json' \
--data-raw '{
    "input": {
        "do-validate" : False,
    }
}'
```

```json RPC Response, Status: 500
{
  "errors": {
    "error": [
      {
        "error-type": "application",
        "error-tag": "operation-failed",
        "error-message": "RemoteDevice{R1}: Confirmed commit failed. illegal reference /orgs/org[name='TESTING-PROVIDER']/traffic-identification/using-networks",
        "error-info": {
          "node-id": "R1",
          "configuration-status": "fail",
          "rollback-status": "complete"
        }
      }
    ]
  }
}
```

### Failed example

Node R1 has failed because of the time delay between confirmed commit and
submitted configuration.

```bash RPC Request
curl --location --request POST 'http://localhost:8181/rests/operations/uniconfig-manager:commit' \
--header 'Accept: application/json' \
--header 'Content-Type: application/json' \
--data-raw '{
    "input": {
    }
}'
```

```json RPC Response, Status: 500
{
  "errors": {
    "error": [
      {
        "error-type": "application",
        "error-tag": "operation-failed",
        "error-message": "RemoteDevice{R1}: time delay between confirmed and confirming commit was greater than 300 seconds, the configured changes were rolled back.",
        "error-info": {
          "node-id": "R1",
          "configuration-status": "fail",
          "rollback-status": "complete"
        }
      }
    ]
  }
}
```

### Failed example

Node R1 has failed due to incorrect configuration.

```bash RPC Request
curl --location --request POST 'http://localhost:8181/rests/operations/uniconfig-manager:commit' \
--header 'Accept: application/json' \
--header 'Content-Type: application/json' \
--data-raw '{
    "input": {
    }
}'
```

```json RPC Response, Status: 500
{
  "errors": {
    "error": [
      {
        "error-type": "application",
        "error-tag": "operation-failed",
        "error-message": "Supplied value \"GigabitEthernet0/0/0/1ghjfhjfhjfghj\" does not match required pattern \"^(([a-zA-Z0-9_]*\\d+/){3,4}\\d+)|(([a-zA-Z0-9_]*\\d+/){3,4}\\d+\\.\\d+)|(([a-zA-Z0-9_]*\\d+/){2}([a-zA-Z0-9_]*\\d+))|(([a-zA-Z0-9_]*\\d+/){2}([a-zA-Z0-9_]+))|([a-zA-Z0-9_-]*\\d+)|([a-zA-Z0-9_-]*\\d+\\.\\d+)|(mpls)|(dwdm)$\"",
        "error-info": {
          "node-id": "R1",
          "configuration-status": "fail",
          "rollback-status": "complete"
        }
      }
    ]
  }
}
```

### Failed example

Node R1 has lost connection.

```bash RPC Request
curl --location --request POST 'http://localhost:8181/rests/operations/uniconfig-manager:commit' \
--header 'Accept: application/json' \
--header 'Content-Type: application/json' \
--data-raw '{
    "input": {
    }
}'
```

```json RPC Response, Status: 502
{
  "errors": {
    "error": [
      {
        "error-type": "application",
        "error-tag": "southbound_no_connection",
        "error-message": "Unified Mountpoint not found.",
        "error-info": {
          "node-id": "R1",
          "configuration-status": "fail"
        }
      }
    ]
  }
}
```

### Failed example

Node R1 has failed because of incorrect configuration. In this case validation,
confirm-commit and auto-rollback are disabled.

Since auto-rollback is disabled, configuration of R1 was successful. However,
this can only be done if the validation and confirm-commit phases were
successful or skipped; otherwise configuration of the R1 device would also fail.

```bash RPC Request
curl --location --request POST 'http://localhost:8181/rests/operations/uniconfig-manager:commit' \
--header 'Accept: application/json' \
--header 'Content-Type: application/json' \
--data-raw '{
    "input": {
        "do-rollback" : false,
        "do-validate" : false,
        "do-confirmed-commit" : false
    }
}'
```

```json RPC Response, Status: 500
{
  "errors": {
    "error": [
      {
        "error-type": "application",
        "error-tag": "operation-failed",
        "error-message": "RemoteDevice{R1}: RPC during tx failed. Error messages: [/alias[name='^new']/expansion is not configured]",
        "error-info": {
          "node-id": "R1",
          "configuration-status": "fail"
        }
      }
    ]
  }
}
```

### Failed example

Configuration of nodes R1 and R2 have been modified in the transaction, and both
are in sync with the actual state on the device. Connection to node R2 has been
lost. RPC input has the flag to skip unreachable nodes set to `true`.

The result of the commit RPC describes success for node R1 and includes a list
of unreachable nodes.

```bash RPC Request
curl --location --request POST 'http://localhost:8181/rests/operations/uniconfig-manager:commit' \
--header 'Accept: application/json' \
--header 'Content-Type: application/json' \
--data-raw '{
    "input": {
        "skip-unreachable-nodes": true
    }
}'
```

```json RPC Response, Status: 200
{
  "output": {
    "unreachable-nodes": [
      "R2"
    ]
  }
}
```
