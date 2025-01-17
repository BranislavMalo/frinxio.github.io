---
label: UniConfig NETCONF
order: 8000
---

# UniConfig NETCONF

## Overview

NETCONF is an Internet Engineering Task Force (IETF) protocol used for
configuration and monitoring of devices in a network. It can be used to
“create, recover, update, and delete configurations of network devices”.
The base NETCONF protocol is described in
[RFC-6241](https://tools.ietf.org/html/rfc6241).

NETCONF operations are overlaid on the Remote Procedure Call (RPC) layer
and may be described in either XML or JSON.

## NETCONF southbound plugin

### Introduction to southbound plugin and netconf-connectors

The NETCONF southbound plugin is capable of connecting to remote NETCONF
devices and exposing their configuration/operational datastores, RPCs
and notifications as MD-SAL mount points. These mount points allow
applications and remote users (over RESTCONF) to interact with the
mounted devices.

In terms of RFCs, the southbound plugin supports:

- Network Configuration Protocol (NETCONF) -
    [RFC-6241](https://tools.ietf.org/html/rfc6241)
- NETCONF Event Notifications -
    [RFC-5277](https://tools.ietf.org/html/rfc5277)
- YANG Module for NETCONF Monitoring -
    [RFC-6022](https://tools.ietf.org/html/rfc6022)
- YANG Module Library -
    [draft-ietf-netconf-yang-library-06](https://tools.ietf.org/html/draft-ietf-netconf-yang-library-06)

NETCONF is fully model-driven (utilizing the YANG modelling language) so
in addition to the above RFCs, it supports any data/RPC/notifications
described by a YANG model that is implemented by the device.

By mounting of NETCONF device a new netconf-connector is created. This
connector is responsible for:

- keeping state of NETCONF session between NETCONF client that resides
    on FRINX UniConfig distribution and NETCONF server (remote network
    device)
- sending / receiving of NETCONF RPCs that are used for reading /
    configuration of network device
- interpreting of NETCONF RPCs by mapping of their content using
    loaded device-specific YANG schemas

There are 2 ways for configuring a new netconf-connector: NETCONF or
RESTCONF. This guide focuses on using RESTCONF.

### Spawning of netconf-connectors while the controller is running

To configure a new netconf-connector (NETCONF mount-point) you need to
create a node in configuration data-store under 'topology-netconf'.
Adding of new node under NETCONF topology automatically triggers
data-change-event that at the end triggers mounting process of the
NETCONF device. The following example shows how to mount device with
node name 'example' (make sure that the same node name is specified in
URI and request body under 'node-id' leaf).

```bash
curl -X PUT \
  http://127.0.0.1:8181/rests/data/network-topology:network-topology/topology=topology-netconf/node=example \
  -d '{
      "node": [
          {
              "node-id": "example",
              "netconf-node-topology:host": "192.168.1.100",
              "netconf-node-topology:port": 22,
              "netconf-node-topology:tcp-only": false,
              "netconf-node-topology:username": "test",
              "netconf-node-topology:password": "test"
          }
      ]
  }'
```

This spawns a new netconf-connector with name 'example' which tries to
connect to the NETCONF device at '192.168.1.100' and port '22'. Both
username and password are set to 'test' and SSH is used as channel for
transporting of NETCONF RPCs (if 'tcp-only' leaf is set to 'true',
NETCONF application protocol is running directly on top of the TCP
protocol).

Right after the new netconf-connector is created, NETCONF layer writes
some useful metadata into the operational data-store of MD-SAL under the
network-topology subtree. This metadata can be found at:

```bash
curl -X GET \
  http://127.0.0.1:8181/rests/data/network-topology:network-topology/topology=topology-netconf/node=example?content=nonconfig
```

Information about connection status, device capabilities, etc. can be
found there.

You can check the configuration of device by accessing of
'yang-ext:mount' container that is created under every mounted NETCONF
node. The new netconf-connector will now be present there. Just invoke:

```bash
curl -X GET \
  http://127.0.0.1:8181/rests/data/network-topology:network-topology/topology=topology-netconf/node=example/yang-ext:mount?content=config
```

The response will contain the whole configuration of NETCONF device. You
can fetch smaller slice of configuration using more specific URLs under
'yang-ext:mount' too.

### Authentication with private/public key

This type of authentication is used when you want to connect to the
NETCONF device via private/public key, it is necessary to save public
key into device, then put private key into UniConfig and when trying to
configure NETCONF mount-point to connect via ssh key and not password.

To accomplish that, follow these steps :

**1.** Generate private/public key-pair on your local machine

```
$ ssh-keygen -b 1024 -t rsa -f sshkey -m pem
```

**2.** Change .pub format into .bin format

```
$ cat sshkey.pub | cut -f 2 -d ' ' | base64 -d > sshkey.bin
```

**3.** Copy public key into device directory. Password of the device
will be required.

```
scp asr_sshkey.bin cisco@192.168.1.216:disk0:/
```

**4.** (Optional) Check if the public key is on device

```
$ ssh cisco@192.168.1.216
(password)

#dir

(you will see this)

    Directory of /misc/scratch
   46 -rw-r--r-- 1   151 Feb 21 20:10 asr_sshkey.bin
   30 -rw-rw-rw- 1   810 Oct 31 08:58 cvac.log
 8178 drwxr-xr-x 2  4096 Oct 31 08:56 kim
   13 -rw-r--r-- 1  1438 Oct 31 08:53 envoke_log
 8179 drwxr-xr-x 2  4096 Jun 27  2019 crypto
   42 -rw-r--r-- 1  1524 Oct 31 08:57 status_file
16354 drwxr-xr-x 2  4096 Jul 23  2019 nvgen_traces
   12 drwxr-xr-x 2  4096 Jun 27  2019 core
   15 lrwxrwxrwx 1    12 Jun 27  2019 config -/misc/config
   11 drwx------ 2 16384 Jun 27  2019 lost+found
16353 drwxr-xr-x 8  4096 Oct 31 08:58 ztp
   14 -rw-r--r-- 1 93861 Oct 31 08:53 pnet_cfg.log
   43 -rwx------ 1   490 Oct 10 09:19 initial_configuration.txt
 8177 drwx------ 2  4096 Jun 27  2019 clihistory
```

**5.** Import public key to device

```
crypto key import authentication rsa disk0:/asr_sshkey.bin
```

**6.** Log in with private key to device NETCONF subsystem. Passphrase
for key will be required.

```
$ ssh -i ./asr_sshkey cisco@192.168.1.216 -s netconf

If it is not possible use optional parameter
$ ssh -o "IdentitiesOnly=yes" -i ./asr_sshkey cisco@192.168.1.216 -s netconf
```

**7.** Start UniConfig and insert keystore with private key into it.

**RPC request:**

```
REST
    PUT
URL
    http://localhost:8181/rests/operations/netconf-keystore:add-keystore-entry
HEAD
    Accept
        application/json
    Content-Type
        application/json

BODY
{
    "input":{
        "key-credential":[
            {
                "netconf-keystore:key-id":"sshkey",
                "netconf-keystore:private-key":"-----BEGIN RSA PRIVATE KEY-----
Proc-Type: 4,ENCRYPTED
DEK-Info: AES-128-CBC,0C4CB5DDAAF81007B4178A05A3CEDF60

2URiEEton5ifZnq2IY4Tn2FhQynkXCd6RxV7rPUYbCvlDo2W7GMMUTPbXNchMOgp
YXuDA/MSUx7lS7K8OHHYmx/dhubnqdfEM3r7LmHDdmee4Rc86xWYTVUktVzZDBEu
pGNs2aL/2wcFIgB3twaBvpqlpNcdIdzRkXGks34MdU/NwIYWP0wxik2Toku3Upfk
knaF+nchAbSCCxv3qmJ1w1/MQq4r6CnTSA9Dl+SVChLvi5EdcjHrOmqfUS6m6k8I
upRIQh4AJ2cl+88yxAsHFJHFUSolcEE7ckrkSfLUYhWYXZ6w8mTw29ocUE0BeQm6
NjseVNVPMrprvTMQUpmNNk5NOApsdQPilDbM8OxHTvGNv0qzlU/dB+FSvkfe9ThZ
YnXaXEPh4VEPPyCyN/pJF+7wmTxEUmabdEpLAz+AEKvq/LHkhWqd2Ep8keDfHpSk
hkTXEc5W/PS0+G7wePIaVC4T++vO37f3YkKmwd2X2bqOQaVa1ddcMO/FSenTZFBc
PbyD8RIIU1rScdHan+BOCgk6h58pvqWHpPLNojQXC0t3ricFRpFBlMGAD/N3F9IP
0NmptxwODio1L8BztKdDosekpPy/tV3M2kWdFlqqbKQtnGk6afyr4YIufJ/KQFfe
d0/FEKtn1rTTkQDbmwmLoFFOycRBEyE2PcmGTCndySL6kLzUjBWrEu5S6cHgqTTg
tbxh/nhw92RHwXkR6/87HRVpjB3gpbDoRvYibwUximOQna/2OHMFDjrfB7Uz5zvV
tuwnQbgswmiIvjISLRNNlHh3GuCHw7ZNowenHheX8vzLsGeW1iywqsVt/H1AYFx4
-----END RSA PRIVATE KEY-----",
                "netconf-keystore:passphrase":"iosxr"
            }
        ]
    }
}
```

**8.** Create mount-point with key-id

**RPC request:**

```
REST
    PUT
URL
    http://localhost:8181/rests/data/network-topology:network-topology/topology=topology-netconf/node=iosxr
HEAD
    Accept
        application/json
    Content-Type
        application/json

BODY
{
    "node" :
    {
        "node-id" : "iosxr",
        "netconf-node-topology:host": "192.168.1.216",
        "netconf-node-topology:port": 830,
        "netconf-node-topology:keepalive-delay": 0,
        "netconf-node-topology:tcp-only": false,
        "netconf-node-topology:key-based": {
            "username": "cisco",
            "key-id": "sshkey"
        }
    }
}
```

**Delete public key**

Login to device, remove rsa public key and after that, it is also
possible to delete key from device directory.

```
$ ssh cisco@192.168.1.216
(password)

#crypto key zeroize authentication rsa
(confirm it with 'yes')

- at this point, it can be enabled again

#dir
#delete disk0:/sshkey.bin
```

### PKI Data persistence in NETCONF

* PKI data is used for authentication of NETCONF sessions with the provided RSA private key.
  The corresponding public key must be stored on the device side.
* Keys are identified using a unique 'key-id'. This key identifier can be specified in the
  NETCONF installation request.
* Keys can be managed using the 'remove-keystore-entry' and 'add-keystore-entry' operations.
  These RPC calls are part of the UniConfig transaction. Changes are not applied until
  they are committed by the user or the immediate commit model is used to invoke the operation.
* Keys are stored in the UniConfig database. In a clustered environment, all nodes
  share the same set of keys.

#### Registration of the new key

The following request demonstrates how to register a new RSA private key with a key-id of 'key1'.
The private key must be specified in the PKCS#8 format. The passphrase is optional and must
be specified only if the private key is encrypted.

Multiple keys can be registered at once if the user provides a list of the 'key-credential' in the input.

```bash

```bash Registration of the new key
curl --location 'http://127.0.0.1:8181/rests/operations/netconf-keystore:add-keystore-entry' \
--header 'Content-Type: application/json' \
--data '{
    "input": {
        "key-credential": [
            {
                "key-id": "key1",
                "private-key": "-----BEGIN RSA PRIVATE KEY-----
MIIEpAIBAAKCAQEAyx3eACAkPMRGTKERfjik+hY1fCoY0FEYVshPvqigjILLlFD6
1phD3Az20gSdrFy6l3x+7IS0Iqy4yhAuj3uSPoE1HWdTxDBG1TpFxNv0kGhOkxwW
X5teB7/wViqYFu7niCvQHP3+cn+bhJhGN71h9hfYXUzkL4/ZP8Ojt9k+cwADgMy1
qvoGlYV7I7MbhI3EHSA8R8/a83rqwRk3Q6pfAu1Dko8ypsBe6b1TWFIHJCoHmIwz
sxbHictzuZKiGuVSb5+dwxH3h3AhV8zjjttwyhCn+220VNfsL49E/1rP+1qODz5H
xIQbIhyoxmfVnwtFl0tQO7EkpctruSKrX7eq8QIDAQABAoIBAQC/7aYDyltTjEe9
SwmDkrTZl7jMbd7qoWZ1QmyQBTlH7KO/r1GHC3Q774ge5buzzh+G21330mGAzmMC
lI3aiPKk22S8NxHhNZkJDTo2DX2oItf0jDrb9qzAd+77DN6P9QmVE4lgi+UufkQO
4/Cd5Pc3GksZxnlsM+oeJZiDNeWbtXxI+DVGf8goQGpsGLq6rzWMGPnY6PFXyV+O
zdhWYAZP0qmXNE0BoMqG0AmUYN9O53Mxx4rYeEAF/+znPOatu9FO9W8IbUH23Exv
JuopiK+8BohKTVQ2Tcuqoq/YZYEbkpBuKdqiao+QOZ4LHKoJ3VnWDgK0xPjR8sI0
z5xGFE0tAoGBAPb0O5hNat1mGbIlad3pBMd2Nfd6qWUuusT9DP3JeCYRn7Tu6rbq
Qvvmeamoe0OWaEzS8PfRSOrlL+GPyB98rHiFkvc2ePu9C3BH3yV1z51kOPCH9KKY
nZHeK4mA/8nQEjDuHR/3bEjZC2J78Rnj00O9JKZDW3k3+poWcZ9f+1pDAoGBANKO
jq7WrqeDT+Dm01nPut+1lYtyMFOce8JNryyKnEMrpRDlEwbu6Wvk3YuuajznwLQ2
7ydUCBtDJJkEPmAhZI38439lyp4fzHFrQeptjEClRJkb4qv36+IcUS59c/MNuPaC
t6FYTjHXZZriTIvXkDWYMPALcV9i3vGb3xTlepS7AoGBAPKe4snwDXS08bvfHBKj
80kPr1eTkEdTULVmM9RGkp88I178d2b74pFfTtpLJ8cwRwprF8kxOWVlg9QkkaW1
tDC4crokL1qL0WgfhHFmPHJSW8qcl9EDBZOg5b8zGJqqrKSb28tjJ2SusIuyXx5j
gVUEx7P9ayo9wByQvlKpVuXHAoGATrhJlAhjZ/FqDdV+sxc88KJ89JOOidP5WR2e
HnL8FQjeP8DFKlRsJJB+W9irk3W50Caxpux902N47VRu1ZXmeEdR7rFp3VBaKRVG
oZSvWQPw76VDS7P3FqQrncv9a6N3wYIBkWroYS38qLlukOHY4pCxyy0cB+N5Iq95
6eAZwj0CgYAz9MWvU/UMnWl958e4Al9Na0PwRWL68Q9pzCkczXODh9/ZqAEMBWUh
wxRzmXqTmesqqDR7rX/sv2qStEOy4sUrZPh49U+9G2xl2OScXMKnjHNu2A7QdbVm
KBhDnvxKaos9nXJrezxThvyA7qD0F4ulBw3jg1nYr4z+oriqxDHKEw==
-----END RSA PRIVATE KEY-----",
                "passphrase": "pa$$word"
            }
        ]
    }
}'
```

#### Removing of the existing key

The following example shows how to remove the existing key 'key1' from UniConfig.
It is possible to remove multiple keys at once.

```bash Removing of the existing key
curl --location 'http://127.0.0.1:8181/rests/operations/netconf-keystore:remove-keystore-entry' \
--header 'Content-Type: application/json' \
--data '{
    "input": {
        "key-id": [
            "key1"
        ]
    }
}'
```

#### Reading list of the existing keys

The following example shows how to read list of the existing keys from UniConfig.

```bash Reading list of the existing keys
curl --location 'http://127.0.0.1:8181/rests/data/netconf-keystore:keystore/key-credential'
```

```json Sample response
{
  "key-credential": [
    {
      "key-id": "key1",
      "passphrase": "UPII5ErGsvPB7L+2OfI4xQ==",
      "private-key": "kjTlzs/EpFAQA6xLjmye5uvWdUtpQyD0oQKan49EIdkweABSFY8QQgl8spXDRWBAjoyyfuwKr0Kp5/EnScFIPBmjaPUDpVB5YMfT1sMEGFP/84kLtsaU1zQaLvFLMmJW4SgCKTe/9sTDO/oj3IfOgH+jT070hOgD8NHvQPTSHvQzTmiuNDjMXtkZDVwsA0+lWyL+GPs9tbpgggVuxm8VO+uWuNFqiRUEjfNl7uV+gg7MB/IWiaLqMkUNBNNYrMN1bhfZgtt81I/i/dw/LaJcY64Fy48QwYAv8+UHWh1WczhZGhLYgbNVSLDkYv5ffVMnfLipw8hqT8vrWVlIQNmeeP0IS7ApmXkFueU+xq10l+DaicoPVMhm6IEZjn8RgJhoX/u4vKcVZbMmzDrHfgDNo1MfddgxNsUr/MyqIbZvHXjsCRZWWziXmE7dTl3WWueEnXUmg5rmxxNTg4Bygkw7azDWDshwGnoteRViENZwlaFd3U32GY1RREvoA+3EHYL+0BcLXF9yxlXwNgigWfSyBqEeJFhGpwHaQPgpV5Z098I0Lk1mbg0c13K61Sdv9ryCjPPl7o1hfV4KCrELW+/5S7SeSwIL5WU77tHFN0ym5OQjlhkMdxIdLod3CcgbCWz3XmXx7yXn94OX1QMaSQnracWTCkjey4qA4o8BD4DzsVvP0DSTfEtFXmaPvXLJMlIP7G6P95o9t7TQoZOUuNYFtyYoejnmsBQrYJ4hqpQO96LUoMxgwYQyQVVdJdEzEp/uuq6xePfK+WGQ5WUucrGoRYq3gD8GezOMvRtQk6QppSgKv5M0TyV/jjs1uzV8np693wMtl9sjKaU75gV6OYb4Za2YCLExRRyStdyBJj7hN1ctYwxXDSVyJLl3JJ5yWmK/RHW/82J1si3VOe+eVKURo+D3/KWvJKTzTklyta0a86qbAB9xoF5nroRK7AyesxwPJhSF7H+RkemFT2O6/eg1DUDRRWUN7YJqbwpNWWgIlsM0ejpAh9jbkBqNDAfgYdr1Ld6BF3BDShfZRitwpXsOhGf8Bbc4JyOcPgVkMdtCOP/VsJsn+9UShjZK4RvN/DWc/e74tYfrdBr09o7hHMC6YaGn8Dfh7OPjpL3NvFbkTGYS5kGqBze14EIB+Ioh/1parymTfMzNxiOfrrj1kZkqInS/a2IcRcA25jc7xeaOql5sB5jBPDUd5yyfFwimkCqJnves+r0e7mr0znqj3/rnoJWwCgVdPfLGY3+o3wTX+ZpNE2N+majAv29kxiMNUy+MP5G55lvMh5U0clBHb0JZfta1rWMntkSIROXyMKdGKR9EWf4FIRjEgWeagU2xKdO/ccged4KXBAxmMDMiQueN+EVBRYA+qPj2ttVEgyJXPMZHhULucExlwGw1SaMYOgyTo0gGkuxi7EknBN0qrOZrjMQVtYeMsvTDjJ7GlABH3RlEL8wUTWAXA7b6GsNxj4GNlGYMxFIQwNZT4dbuBp2V8lQu4oqoSBCk2lAMYotaKXi26xqKCDKofwlzAvxHQFfJ+nwXPQ/5qKFR+F+rXT4suuWYeA9In2RBMxOzOIuvXNC1uOQRs3N6ORiNfS6Y7dF3HUHLAwud5cO3xAdFFEXmHBWvz7TJ1MzkYu5ggkNhSaFBGncvWZKuXOjfwvGXVaxZbRU3ORfEz56FC2Mg8Mx1aOO9m9mD7XssrKUZwSqKgrnol0sknva2Tb7+8FleOf7PgzkkDdb8r4y/uNqybjAL7XzI4JCUo66U2eRCGJsGJffMfeJN5vihbbt3Uq6+1FTvSxZa4iRpPeKMk1qLAtZRO78CvRD1/igcgq/0h7tFtiQtGdhkm/hTM55OaeuI1f3BmbIXY3SJVwMOKqkdiSMTDlnRR2hmoN8AFyYzqDK5ZyGmLgwnNO7tuDD7+TYBF8qK4GE6l3pfJ0kmeMeu0UVYrtqaKncMkjxvmaS5lGu2B8wNNa5ka9cS/bCJyaU39lred8C+bhx21oMajDrUaURIBCf6v5QC+X96XEGAB96sNssfEZdlS1zMGaSqmSr3LbFKsm4hI3CZ2NIK+qXKAXO9nCeHxIHMg80vBEXmhbbKviBzcsYUj0lA0Qr0WXz9qe0YjDwUq6RwGP4nHM62jHWEgd87FiYX/+RBiKa3PwJ9OU7ax+3PkOgJZLcjQKiDiHkhPN9ocOnEvQqC8u53qRTG3sSm0AusYri3jgb3IWg5fjOKLT7fLnHdyKwogj+zqfxK"
    }
  ]
}
```

**Note:** Both 'passphrase' and 'private-key' are additionally encrypted by the UniConfig encryption system
to protect confidential data.

### Keepalive settings

If the NETCONF session haven't been created yet, the session is tried to
be established only within maximum connection timeout. If this timeout
expires before NETCONF session is established, underlay NETCONF channel
is closed (reconnection strategy will not be started). After the NETCONF
session has been successfully created, there are two techniques how the
connection state is kept alive:

1.  **TCP acknowledgements** - NETCONF is running on top of the TCP
    protocol that can handle dropped packets by decreasing of window
    size and resending of lost TCP segments. Working TCP connection
    doesn't imply working state of the application layer (NETCONF
    session) - keepalive messages are required too.
2.  **Explicit NETCONF keepalive messages** - Keepalive messages test
    whether NETCONF server is alive - server responds to keepalive
    messages within NETCONF RPC timeout.

If TCP connection is dropped or NETCONF server doesn't respond within
keepalive timeout, NETCONF launches reconnection strategy. To summarize
it all, there are 3 configurable parameters that can be set in
mount-request:

1.  **Initial connection timeout [seconds]** - Specifies timeout in
    milliseconds after which initial connection to the NETCONF server
    must be established. By default, the value is set 20 s.
2.  **Keepalive delay [seconds]** - Delay between sending of keepalive
    RPC messages to the NETCONF server. Keepalive messages test state of
    the NETCONF session (application layer) - whether remote side is
    able to respond to RPC messages. Default keepalive delay is 120
    seconds.
3.  **Request transaction timeout [seconds]** - Timeout for blocking RPC
    operations within transactions. Southbound plugin stops to wait for
    RPC reply after this timeout expires. By default, it is set to 60 s.

Example with set keepalive parameters at creation of NETCONF mount-point
(connection timeout, keepalive delay and request timeout):

```
{
  "node": [
      {
          ...
          "netconf-node-topology:session-timers": {
              "netconf-node-topology:initial-connection-timeout-": 5,
              "netconf-node-topology:keepalive-delay": 60,
              "netconf-node-topology:request-transaction-timeout": 10
          }
      }
  ]
}
```

### Reconnection strategy

Reconnection strategies are used for recovering of the lost connection
to the NETCONF server. The behaviour of the reconnection can be
described by 3 configurable mount-request parameters:

1.  **Maximum number of connection attempts [count]** - Maximum number
    of initial connection retries. This is used when no connection is 
    yet established for given node inside uniconfig transaction; when it is reached, the NETCONF won't
    try to connect to device anymore. By default, this value is set to 1, non-positive value or null is interpreted as infinity.
2.  **Maximum number of reconnection attempts [count]** - Maximum number
    of reconnection retries. This is used when connection was already successfully established and ongoing (not yet closed),
    but it dropped (from device side); when it is reached, the NETCONF won't try
    to reconnect to device anymore. By default, this value is set to 0, non-positive value or null is interpreted as infinity.
3.  **Initial timeout between attempts [seconds]** - The first
    timeout between reconnection attempts in milliseconds. The default
    timeout value is set to 2000 ms.
4.  **Reconnection attempts multiplier [factor]** - After each reconnection attempt, the
    delay between reconnection attempts is multiplied by this factor. By
    default, it is set to 1.5. This means that the next delay between
    attempts will be 3 s, then it will be 4,5 s, etc.

Example with set reconnection parameters at creation of NETCONF
mount-point - maximum connection attempts, initial delay between
attempts and sleep factor:

```
{
  "node": [
      {
          ...
          "netconf-node-topology:session-timers": {
              "netconf-node-topology:max-connection-attempts": 10,
              "netconf-node-topology:max-reconnection-attempts": 10,
              "netconf-node-topology:between-attempts-timeout": 8,
              "netconf-node-topology:reconnection-attempts-multiplier": 1.0
          }
      }
  ]
}
```

### Local NETCONF cache repositories

The netconf-connector in OpenDaylight relies on
'ietf-netconf-monitoring' support when connecting to remote NETCONF
device. The 'ietf-netconf-monitoring' feature allows netconf-connector
to list and download all YANG schemas that are used by the device. These
YANG schemas are afterwards used by NETCONF southbound plugin for
interpretation of RPCs. The following rules apply for maintaining of
local NETCONF cache repositories:

-   By default, for each device type, the separate local repository is
    prepared.
-   All NETCONF repositories are backed up by separate sub-directory
    under 'cache' directory of UniConfig Distribution.
-   NETCONF device types are distinguished by unique set of YANG source
    identifiers - module names and revision numbers. For example, if 2
    NETCONF devices differ only in revision of one YANG schema, these
    NETCONF devices are recognized to have different device types.
-   Format of the name of generated NETCONF cache directory at runtime
    is 'schema\_id', where 'id' represents unique integer computed from
    hash of all source identifiers. This generation of cache directory
    name is launched only at mounting of new NETCONF device and only if
    another directory with the same set of source identifiers haven't
    been registered yet.
-   You can still manually provide NETCONF cache directories with
    another format before starting of UniConfig Distribution or at
    runtime - such directories don't have to follow 'schema\_id' format.

The NETCONF repository can be registered in 3 ways:

1.  Implicitly by mounting of NETCONF device that has NETCONF monitoring
    capability and another devices with the same type hasn't already
    been mounted.
2.  At booting of FRINX UniConfig distribution, all existing
    sub-directories of 'cache' root directory are registered as separate
    NETCONF repositories.
3.  At runtime, by invocation of 'schema-resources:register-repository'
    RPC.

Already registered schema repositories can be listed using following
request:

```bash
curl -X GET \
  http://127.0.0.1:8181/rests/data/schema-resources:odl-nodes?content=nonconfig
```

It should return list of ODL nodes in cluster with list of all loaded
repositories. Each repository have associated list of source
identifiers. See the following example of GET request output:

```
{
    "odl-nodes": {
        "odl-node-state": [
            {
                "odl-node-id": "127.0.0.1:2550",
                "loaded-repository": [
                    {
                        "repository-name": "schema_1757284974",
                        "source-identifier": [
                            {
                                "module-name": "module1",
                                "module-revision": "2015-11-09"
                            },
                            {
                                "module-name": "module2",
                                "module-revision": "2015-05-14"
                            }
                        ]
                    },
                    {
                        "repository-name": "schema_2018244966",
                        "source-identifier": [
                            {
                                "module-name": "moduleX",
                                "module-revision": "2019-01-09"
                            },
                            {
                                "module-name": "moduleY",
                                "module-revision": "2019-01-01"
                            }
                        ]
                    }
                ]
            }
        ]
    }
}
```

### Local Netconf default cache repository

Before booting of FRINX UniConfig, the user can put the 'default'
repository in the ‘cache’ directory. This directory should contain the
most frequently missing sources. As mentioned above, if the device
supports ‘ietf-netconf-monitoring’ and there is no directory in the
'cache' with all sources that the device requires, then NETCONF will
generate directory with name ‘schema\_id’, where ‘id’ represents unique
integer. The generated repository may not contain all required schemas
because device may not provide them. In such case, the missing sources
will be searched in the 'default' repository and if sources will be
located there, generated repository will be supplemented by the missing
sources. In general, there are 2 situations that can occur:

1.  **Missing imports**

The device requires and provides a resource which for its work
requires additional resources that are not covered by provided
resources.

2.  **Source that is not covered by provided sources**

The device requires but does not provide a specific source.

**note**
Using the 'default' directory in the 'cache' directory is optional.

### Connecting to a device not supporting NETCONF monitoring

NETCONF connector can only communicate with a device if it knows the set
of used schemas (or at least a subset). However, some devices use YANG
models internally but do not support NETCONF monitoring.
Netconf-connector can also communicate with these devices, but you must
load required YANG models manually. In general, there are 2 situations
you might encounter:

1.  **NETCONF device does not support 'ietf-netconf-monitoring' but it
    does list all its YANG models as capabilities in HELLO message**

This could be a device that internally uses, for example,
'ietf-inet-types' YANG model with revision '2010-09-24'. In the HELLO
message, that is sent from this device, there is this capability
reported as the following string (other YANG schemas can be reported
as capabilities in the similar format):
```
urn:ietf:params:xml:ns:yang:ietf-inet-types?module=ietf-inet-types&amp;revision=2010-09-24
```
The format of the capability string is following:
```
[NAMESPACE]?module=[MODULE_NAME]&amp;revision=[REVISION]
```
-   [NAMESPACE] - Namespace that is specified in the YANG schema.
-   [MODULE\_NAME] - Name of the YANG module.
-   [REVISION] - The newest revision that is specified in the YANG
    schema (it should be specified as the first one in the file).
**note**
Revision number is not mandatory (YANG model doesn't have to contain
revision number) - then, the capability is specified without the
'&amp;' and revision too.
For such devices you have to side load all device YANG models into
separate sub-directory under 'cache' directory (you can choose random
name for this directory, but directory must contain only YANG files of
one device type).

2.  **NETCONF device does not support 'ietf-netconf-monitoring' and it
    does NOT list its YANG models as capabilities in HELLO message**

Compared to device that lists its YANG models in HELLO message, in
this case there would be no specified capabilities in the HELLO
message. This type of device basically provides no information about
the YANG schemas it uses so its up to the user of OpenDaylight to
properly configure netconf-connector for this device.
Netconf-connector has an optional configuration attribute called
'yang-module-capabilities' and this attribute can contain a list of
'yang-module-based' capabilities. By setting this configuration
attribute, it is possible to override the 'yang-module-based'
capabilities reported in HELLO message of the device. To do this, we
need to mount NETCONF device or modify the configuration of existing
netconf-connector by adding the configuration snippet with explicitly
specified capabilities (it needs to be added next to the address,
port, username etc. configuration elements).
The following example shows explicit specification of 6 capabilities:
```
"netconf-node-topology:yang-module-capabilities": {
    "capability": [
        "urn:ietf:params:xml:ns:a?module=module-a&amp;revision=2018-04-09",
        "urn:ietf:params:xml:ns:b?module=module-b&amp;revision=2014-06-09",
        "urn:ietf:params:xml:ns:c?module=module-c&amp;revision=1998-10-19",
        "urn:ietf:params:xml:ns:d?module=module-damp;revision=2018-04-09",
        "urn:ietf:params:xml:ns:e?module=module-e&amp;revision=2017-09-17",
        "urn:ietf:params:xml:ns:f?module=module-f"
    ]
}
```
**Remember to also put the YANG schemas into the cache folder like in
the case 1.**

### Registration or refreshing of NETCONF cache repository using RPC

This RPC can be used for registration of new NETCONF cache repository or
updating of NETCONF cache repository. This is useful when user wants to
add new NETCONF cache repository at runtime of FRINX UniConfig
distribution for device that doesn't support 'ietf-netconf-monitoring'
feature. It can also be used for refreshing of repository contents (YANG
schemas) at runtime.

The following example shows how to register a NETCONF repository with
name 'example-repository'. The name of the provided repository must
equal to name of the directory which contains YANG schemas.

```bash
curl -X POST \
  http://127.0.0.1:8181/rests/operations/schema-resources:register-repository \
  -d '{
    "input": {
      "repository-name": "example-repository"
    }
  }'
```

If the repository registration or refreshing process ends successfully,
the output contains just set 'status' leaf with 'success' value:

```
{
    "output": {
        "status": "success"
    }
}
```

On the other side, if the directory with input 'repository-name' does
not exist, directory doesn't contain any YANG files, or schema context
cannot be built using provided YANG sources the response body will
contain 'failed' 'status' and set 'error-message'. For example,
non-existing directory name produces following response:

```
{
    "output": {
        "status": "failed",
        "error-message": "Repository with name 'example-repository' doesn't exist in file-system under cache directory."
    }
}
```

Constraints:

-   Only the single repository can be registered using one RPC request.
-   Removal of registered repositories is not supported for now.

### Reconfiguring netconf-connector while the controller is running

It is possible to change the configuration of an already mounted NETCONF
device while the whole controller is running. This example will continue
where the last left off and will change the configuration for the
existing netconf-connector after it was spawned. Using one RESTCONF
request, we will change both username and password for the
netconf-connector.

To update an existing netconf-connector you need to send following
request to RESTCONF:

```bash
curl -X PUT \
  http://127.0.0.1:8181/rests/data/network-topology:network-topology/topology=topology-netconf/node=example \
  -d '{
      "node": [
          {
              "node-id": "example",
              "netconf-node-topology:host": "192.168.1.100",
              "netconf-node-topology:port": 22,
              "netconf-node-topology:tcp-only": false,
              "netconf-node-topology:username": "bob",
              "netconf-node-topology:password": "passwd"
          }
      ]
  }'
```

Since a PUT is a replace operation, the whole configuration must be
specified along with the new values for username and password. This
should result in a '2xx' response and the instance of netconf-connector
called 'example' will be reconfigured to use username 'bob' and password
'passwd'. New configuration can be verified by executing:

```bash
curl -X GET \
  http://127.0.0.1:8181/rests/data/network-topology:network-topology/topology=topology-netconf/node=example?content=config
```

With new configuration, the old connection will be closed and a new one
established.

### Destroying of netconf-connector

Using RESTCONF one can also destroy an instance of a netconf-connector -
NETCONF connection will be dropped and all resources associated with
NETCONF mount-point on NETCONF layer will be cleaned (both CONFIGURATION
and OPERATIONAL data-store information). To do this, simply issue a
request to following URL:

```bash
curl -X DELETE \
  http://127.0.0.1:8181/rests/data/network-topology:network-topology/topology=topology-netconf/node=example
```

The last element of the URL is the name of the mount-point.

NETCONF TESTTOOL
-----------------

### Testtool overview

NETCONF testtool is the Java application that:

-   Can be used for simulation of 1 or more NETCONF devices (it is
    suitable for scale testing).
-   Uses core implementation of NETCONF NORTHBOUND server.
-   Provides broad configuration options of simulated devices.
-   Supports YANG notifications.

NETCONF testtool is available at netconf repository of ODL
(<https://git.opendaylight.org/gerrit/admin/repos/netconf under
'netconf/tools/netconf-testtool' module. After building of this module
using maven (just invoke command 'mvn clean install' in this directory),
the java executable can be found in appeared 'target' directory with
name 'netconf-testtool-[version]-executable.jar' (version placeholder
depends on used release).

Up-to-date NETCONF testtool is also available at frinx artifactory <https://artifactory.frinx.io/>.  
Up-to-date NETCONF testtool is also available at DockerHub as Docker Image.  

### Starting of the NETCONF testtool

After NETCONF testtool has been built, it can be used using the
following command:

```
java -Djava.security.egd=file:/dev/./urandom \
     --agentlib:jdwp=transport=dt_socket,server=y,suspend=n,address=127.0.0.1:8000 \
     -Xmx1G \
     -jar netconf-testtool-[version]-executable.jar \
     --ssh SSH \
     --md-sal MD-SAL \
     --device-count DEVICE-COUNT \
     --starting-port STARTING-PORT \
     --schemas-dir SCHEMAS-DIR \
     --debug ENABLE-DEBUGGING
```
Please see used fields and placeholders explained below ...

Multi line example with values replacing placeholders e.g.: 
```
java -Djava.security.egd=file:/dev/./urandom \
     -agentlib:jdwp=transport=dt_socket,server=y,suspend=n,address=127.0.0.1:8000 \
     -Xmx1G \
     -jar netconf-testtool-5.1.5-20230310.105106-8-executable.jar \
     --ssh true \
     --md-sal true \
     --device-count 10 \
     --starting-port 17830 \
     --schemas-dir schema-1987709419 \
     --debug true
```
Usually no need to debug (it can be omitted) and long jar executable can be renamed:
```
java -Djava.security.egd=file:/dev/./urandom \
     -Xmx1G \
     -jar netconf-testtool.jar \
     --ssh true \
     --md-sal true \
     --device-count 10 \
     --starting-port 17830 \
     --schemas-dir schema-1987709419
```
One liner example:
```
java -Djava.security.egd=file:/dev/./urandom -Xmx1G -jar netconf-testtool.jar --ssh true --md-sal true --device-count 10 --starting-port 17830 --schemas-dir schema-1987709419
```
The following snippet shows output from successfully simulated NETCONF
device (notice the last line that shows hint, on which TCP ports
simulated devices have been started):
```12:33:32.840 [main] INFO  o.o.n.t.tool.NetconfDeviceSimulator - Starting 10, SSH simulated devices starting on port 17830
12:33:32.852 [main] INFO  o.o.n.t.tool.NetconfDeviceSimulator - Loading models from directory.
12:33:42.490 [main] INFO  o.o.n.t.tool.NetconfDeviceSimulator - using PersistentMdsalOperationProvider.
12:33:42.491 [main] INFO  o.o.n.t.tool.NetconfDeviceSimulator - data will be persisted across sessions
12:33:42.575 [main] INFO  o.a.s.c.u.s.b.BouncyCastleSecurityProviderRegistrar - getOrCreateProvider(BC) created instance of org.bouncycastle.jce.provider.BouncyCastleProvider
12:33:42.625 [main] WARN  io.netty.bootstrap.ServerBootstrap - Unknown channel option 'SO_BACKLOG' for channel '[id: 0x3fb75375]'
...
12:33:42.714 [main] INFO  o.o.n.t.tool.NetconfDeviceSimulator - All simulated devices started successfully from port 17830 to 17839
```
You can check occupied ports and netconf-testool process like this:
```
sudo ss -tlp
ps aux | grep java
```


Description of some of the used fields and placeholders:

-   **SCHEMAS-DIR** - Path to the directory that contains YANG schemas
    used for simulation of all NETCONF devices.
-   **DEVICE-COUNT** - Number of NETCONF devices that should be
    simulated at once.
-   **ENABLE-DEBUGGING** - It should be set to 'true', if you want to
    see detailed debugging messages from simulation of NETCONF device
    (for example, received and sent RPC messages); otherwise it should
    be set to 'false' (INFO logging level is used).
-   **STARTING-PORT** - The first TCP port on which the first simulated
    device will listen on - other simulated devices will reserve next
    TCP ports in order by incrementing of this value.
-   **SSH** - It should be set to 'true' if NETCONF session should be
    created on top of SSH session. If it is set to 'false', TCP is used
    as carrier protocol.
-   **MD-SAL** - Whether to use md-sal datastore ('true') instead of
    default simulated datastore ('false').

All configurable parameters can be fetched using help modifier:

```
java -jar netconf-testtool-[version]-executable.jar -h
```
e.g. 
```
usage: netconf testtool [-h] [--edit-content EDIT-CONTENT] [--async-requests {true,false}] [--thread-amount THREAD-AMOUNT]
                        [--throttle THROTTLE] [--auth AUTH AUTH] [--controller-destination CONTROLLER-DESTINATION]
                        [--device-count DEVICES-COUNT] [--devices-per-port DEVICES-PER-PORT] [--schemas-dir SCHEMAS-DIR]
                        [--notification-file NOTIFICATION-FILE] [--initial-config-xml-file INITIAL-CONFIG-XML-FILE]
                        [--starting-port STARTING-PORT] [--generate-config-connection-timeout GENERATE-CONFIG-CONNECTION-TIMEOUT]
                        [--generate-config-address GENERATE-CONFIG-ADDRESS] [--generate-configs-batch-size GENERATE-CONFIGS-BATCH-SIZE]
                        [--distribution-folder DISTRO-FOLDER] [--ssh {true,false}] [--exi {true,false}] [--debug {true,false}]
                        [--md-sal {true,false}] [--md-sal-persistent {true,false}] [--time-out TIME-OUT] [-ip IP]
                        [--thread-pool-size THREAD-POOL-SIZE] [--rpc-config RPC-CONFIG]

Netconf Testtool

Simulates netconf devices:
- one device per port
- can simulate tens of thousands of devices at a time
- supports basic netconf get-config, edit-config, lock, unlock, discard-changes + notifications
- 2 modes:
 - simple: replies with static content to each get-config operation. Edit-config is ignored, filtering doesn't work
- md-sal: full blown basic netconf device. Starts  with  empty  data  store.  Can  persist  data  across sessions or isolate sessions to a
device/port


Example usage:

- Running a single simulated device emulating IOS XR with fully supported netconf datastore. Wipes out datastore for every session

	java -jar ./netconf-testtool-executable.jar --schemas-dir ~/iosxr/ --md-sal true


- Running a single simulated device emulating IOS XR with fully supported netconf datastore. Preserves datastore content across sessions

	java -jar ./netconf-testtool-executable.jar --schemas-dir ~/iosxr/ --md-sal true --md-sal-persistent true


- Running a single simulated device emulating IOS XR with  hardcoded  response  to get-config. Netconf subtree filtering is not working in
this setup

	java -jar ./netconf-testtool-executable.jar --schemas-dir ~/iosxr/ --initial-config-xml-file data.xml --notification-file notif.xml

	Note:  File  data.xml  should  look  like  this:  '<config  xmlns="urn:ietf:params:xml:ns:netconf:base:1.0"><interface-configurations>...
</interface-configurations></config>' 
	Note:   File   notif.xml   should   look   like   this:   '<notifications><notification><times>0</times><delay>0</delay><content><![CDATA
[<eventTime>XXXX</eventTime>]]></content></notification></notifications>' 


named arguments:
  -h, --help             show this help message and exit
  --edit-content EDIT-CONTENT
  --async-requests {true,false}
  --thread-amount THREAD-AMOUNT
                         The number of threads to use for configuring devices.
  --throttle THROTTLE    Maximum amount of async requests that can be open at  a  time, with mutltiple threads this gets divided among all
                         threads
  --auth AUTH AUTH       Username and password for HTTP basic authentication in order username password.
  --controller-destination CONTROLLER-DESTINATION
                         Ip address and port of controller. Must  be  in  following  format  <ip>:<port>  if available it will be used for
                         spawning  netconf  connectors  via  topology  configuration  as   a  part  of  URI.  Example  (http://<controller
                         destination>/restconf/config/network-topology:network-topology/topology/topology-netconf/node/<node-id>)
                         otherwise it will just start simulated devices and skip the execution of PUT requests
  --device-count DEVICES-COUNT
                         Number of simulated netconf devices to spin. This is the number of actual ports open for the devices.
  --devices-per-port DEVICES-PER-PORT
                         Amount of config files generated per port to spoof more devices then are actually running
  --schemas-dir SCHEMAS-DIR
                         Directory containing yang schemas to describe simulated  devices.  Some  schemas e.g. netconf monitoring and inet
                         types are included by default
  --notification-file NOTIFICATION-FILE
                         Xml file containing notifications that should be sent to clients after create subscription is called
  --initial-config-xml-file INITIAL-CONFIG-XML-FILE
                         Xml file containing initial simulatted configuration to be returned via get-config rpc
  --starting-port STARTING-PORT
                         First port for simulated device. Each other device will have previous+1 port number
  --generate-config-connection-timeout GENERATE-CONFIG-CONNECTION-TIMEOUT
                         Timeout to be generated in initial config files
  --generate-config-address GENERATE-CONFIG-ADDRESS
                         Address to be placed in generated configs
  --generate-configs-batch-size GENERATE-CONFIGS-BATCH-SIZE
                         Number of connector configs per generated file
  --distribution-folder DISTRO-FOLDER
                         Directory where the karaf distribution for controller is located
  --ssh {true,false}     Whether to use ssh for transport or just pure tcp
  --exi {true,false}     Whether to use exi to transport xml content
  --debug {true,false}   Whether to use debug log level instead of INFO
  --md-sal {true,false}  Whether to use md-sal datastore instead of default simulated datastore.
  --md-sal-persistent {true,false}
                         Whether to persist data in the md-sal datastore across sessions
  --time-out TIME-OUT    the maximum time in seconds for executing each PUT request
  -ip IP                 Ip address which will be used for creating a  socket  address.It  can either be a machine name, such as java.sun.
                         com, or a textual representation of its IP address.
  --thread-pool-size THREAD-POOL-SIZE
                         The number of threads to keep in the pool, when creating a device simulator. Even if they are idle.
  --rpc-config RPC-CONFIG
                         Rpc config file. It can be used to  define  custom  rpc  behavior, or override the default one.Usable for testing
                         buggy device behavior.

```
Notes:
1. while you use --md-sal true devices are started with empty datastore. You can put initial config via netconf session or via uniconfig operations.
2. --md-sal true --md-sal-persistent true will preserves datastore content across netconf sessions
3. --initial-config-xml-file data.xml overrides --md-sal true and send hardcoded response to get-config.
4. --notification-file notif.xml notification support

### Starting of the NETCONF testtool from Docker Image

1. download docker image using download command
```
docker pull frinx/netconf-testtool
```
2. run it using docker run command
```
docker run -it frinx/netconf-testtool
```
3. you can use options of Netconf test tool starting script
```
-x or --xmx to set Java heap for test tool
-p or --port to set starting port
-d or --devices to set number of simulated devices
-s or --schemas to set schemas folder (required firstly mount schemas folder to docker container using -v)
-i or --init-conf to set initial simulated config to be returned by get-config RPC (required firstly mount this config file to docker container using -v)
--ssh to enable ssh
--md-sal to enable md-sal
--debug to run in debug mode
-h or --help to display help options
--override to override these options to access all netconf test tool options directly
```
**note**  
To set value of xmx is required to separate -x or --xmx option and value of xmx with space.  
When using xmx setting option together with override option, it is required to use xmx setting option first before override.  
All options and their values should be separated with space.  
Setting to enable ssh or md-sal requires only option ssh or md-sal to be present to enable these functions and no other value is required.  
When override is used, then it is not possible to set any other option from simple interface except Java specific options, like xmx and partially also debug.  
After override option there should be Netconf test tool complex interface options.  
If you want to use external files like schemas or init config file, you should first mount them using -v option of docker.  
You should use --publish option to publish ports if you want to use them.  
You can use --rm option after docker run to automatically remove docker container after it is stopped.  

```
docker run --publish 1024-1123:1024-1123 -v ./schemas:/opt/schemas -v ./i.xml:/opt/i.xml -it frinx/netconf-testtool \
--xmx 1G --port 1024 --devices 100 --schemas ./schemas --init-conf ./i.xml --ssh --md-sal --debug
```
4. you can also use help option to show how to use it using -h or --help
```
docker run --rm -it frinx/netconf-testtool --help
```
5. you can also override simple interface of starting script and use all options of test tool directly
```
docker run --publish 1024-1123:1024-1123 -v ./schemas:/opt/schemas -v ./i.xml:/opt/i.xml -v ./notification.xml:/opt/notification.xml \
            -it frinx/netconf-testtool --xmx 1G --override \
            --ssh true \
            --md-sal true \
            --starting-port 1024 \
            --device-count 100 \
            --schemas-dir ./schemas \
            --initial-config ./i.xml \
            --notification-file ./notification.xml \
            --debug true
```
**note**  
If you override starting script options, you should set Java heap size using -x or --xmx option before --override option.  
You could not use any other starting script options except xmx and debug option.  
If you override starting script options, --debug option activate only Java and shell debug mode, but not debug mode of test tool itself.  
If you want to start debug mode of netconf test tool, you should use debug options of Netconf test tool directly.  

6. you can also override docker entrypoint and run netconf test tool by custom way using all options including java options and run like bellow
```
docker run --publish 1024-1123:1024-1123 -v ./schemas:/opt/schemas -v ./i.xml:/opt/i.xml -v ./notifications.xml:/opt/notification.xml \
-it --entrypoint /bin/bash frinx/netconf-testtool
```
and in docker container shell:
```
java -Djava.security.egd=file:/dev/./urandom \
     -Xmx1G \
     -jar netconf-testtool-executable.jar \
     --ssh true \
     --md-sal true \
     --device-count 100 \
     --starting-port 1024 \
     --schemas-dir ./schemas \
     --initial-config ./i.xml \
     --notification-file ./notification.xml \
     --debug true
```
7. in custom way of running test tool there are these files, placed in /opt folder
```
netconf-testtool-executable.jar   file to run netconf test tool
run_netconf_testtool.sh           shell script file to start netconf test tool as default entrypoint
```

### Prepare init config via Uniconfig for simulated netconf-testtool device
1. start Uniconfig - in cache folder there can be present the folder with device yang models to preload them faster
2. start netconf-testool with schema-dir folder of device yang models 
```
java -agentlib:jdwp=transport=dt_socket,server=y,suspend=n,address=127.0.0.1:8000 -Xmx1G -jar testtool.jar --schemas-dir schema-1987709419 --device-count 1 --debug false --starting-port 36000 --ssh true --md-sal true
```
3. install device
```
curl --location --request POST 'http://localhost:8181/rests/operations/connection-manager:install-node' \
--header 'Authorization: Basic YWRtaW46YWRtaW4=' \
--header 'Content-Type: application/json' \
--data-raw '{
    "input": {
        "node-id": "testtool-setup",
        "netconf": {
            "netconf-node-topology:host": "10.19.0.20",
            "netconf-node-topology:port": 36000,
            "netconf-node-topology:keepalive-delay": 5,
            "netconf-node-topology:max-connection-attempts": 1,
            "netconf-node-topology:connection-timeout-millis": 60000,
            "netconf-node-topology:default-request-timeout-millis": 60000,
            "netconf-node-topology:tcp-only": false,
            "netconf-node-topology:username": "admin",
            "netconf-node-topology:password": "admin",
            "netconf-node-topology:sleep-factor": 1.0,
            "uniconfig-config:install-uniconfig-node-enabled": false,
            "uniconfig-config:uniconfig-native-enabled": false
        }
    }
}'
```
4. send init config to the netconf-testtool device via uniconfig
```
curl --location --request PUT 'http://127.0.0.1:8181/rests/data/network-topology:network-topology/topology=topology-netconf/node=testtool-setup/yang-ext:mount/system' \
--header 'Authorization: Basic YWRtaW46YWRtaW4=' \
--header 'Content-Type: application/json' \
--data-raw '{
    "system:system": {
        "identification": {
            "latitude": "31.202149",
            "name": "Site2Branch2",
            "longitude": "-96.666829",
            "location": "Somewhere"
        },
        "users": [
            {
                "name": "admin",
                "login": "shell",
                "role": "admin"
            },
            {
                "name": "my_device",
                "login": "shell",
                "role": "admin"
            }
        ],
        "ssh": {
            "client-alive-count-max": 0,
            "client-alive-interval": 300
        },
        "time-zone": "America/Los_Angeles",
        "services": {
            "ssh": "enabled",
            "sftp": "disabled",
            "www": "enabled"
        },
        "session": {
            "reevaluate-reverse-flow": false,
            "tcp-send-reset": false,
            "check-tcp-syn": false,
            "tcp-secure-reset": false,
            "tcp-adjust-mss": {
                "enable": true,
                "interface-types": "tunnel"
            }
        }
    }
}'
```
### Example of notification file
```
<?xml version='1.0' encoding='UTF-8' standalone='yes'?>

<notifications>
<!-- Notifications are processed in the order they are defined in XML -->
<!-- Notification that is sent only once right after create-subscription is called -->
<notification>
    <!-- Content of each notification entry must contain the entire notification with event time. Event time can be hardcoded, or generated by testtool if XXXX is set as eventtime in this XML -->
    <content><![CDATA[
        <notification xmlns="urn:ietf:params:xml:ns:netconf:notification:1.0">
            <eventTime>XXXX</eventTime>
            <random-notification xmlns="urn:ietf:params:xml:ns:netmod:test">
                <random-content>single no delay</random-content>
            </random-notification>
        </notification>
    ]]></content>
</notification>
<!-- Repeated Notification that is sent 5 times with 2 second delay inbetween -->
<notification>
    <!-- Delay in seconds from previous notification -->
    <delay>20</delay>
    <!-- Number of times this notification should be repeated -->
    <times>5</times>
    <content><![CDATA[
        <notification xmlns="urn:ietf:params:xml:ns:netconf:notification:1.0">
            <eventTime>XXXX</eventTime>
            <random-notification xmlns="urn:ietf:params:xml:ns:netmod:test">
                <random-content>scheduled 5 times 10 seconds each</random-content>
            </random-notification>
        </notification>
    ]]></content>
</notification>
<!-- Single notification that is sent only once right after the previous notification -->
<notification>
    <delay>20</delay>
    <content><![CDATA[
        <notification xmlns="urn:ietf:params:xml:ns:netconf:notification:1.0">
            <eventTime>XXXX</eventTime>
            <random-notification xmlns="urn:ietf:params:xml:ns:netmod:test">
                <random-content>single with delay</random-content>
            </random-notification>
        </notification>
    ]]></content>
</notification>
</notifications>

```


### Increasing the maximum number of opened files

Since NETCONF testtool can be used for simulation of large number of
NETCONF devices, it requires opening a lot of TCP sockets that listen on
different TCP ports. In Linux systems TCP socket is also represented as
file - from this reason such simulations can easily exhaust configured
limit of maximum number of opened files. Then, if the buffering file for
connection cannot be created on time it can cause continuous
reconnection attempts.

Usually, the default soft limit for maximum number of opened files is
set to 1024 (reaching this limit should produce warnings in logging
messages) and hard limit to 4096 (it cannot be exceeded). For setting of
custom soft and hard limits you must modify the following lines in
"/etc/security/limits.conf" file:

```
[user-name] soft nofile 4096
[user-name] hard nofile 10240
```

Replace '[user-name]' by login-name of the user under whom you start
NETCONF test-tool.

You can check the current limits using following commands:

```
ulimit -Hn
ulimit -Sn
```

Soft limit '4096' and hard limit '10240' should be enough, but it also
depends on occupation by other applications and operating system too).

**note**
Configured value should not reach the one that applies for all users -
"cat /proc/sys/fs/file-max".

see also /etc/sysctl.conf:
```
#
## /etc/sysctl.conf
## Increase Outbound Connections
## Good for a service mesh and proxies like 
## Nginx/Envoy/HAProxy/Varnish and applications that
## need long-lived connections.
## Careful not to set the range wider as you will impact
## running application ports in heavy usage situations.
net.ipv4.ip_local_port_range = 12000 65535

## Increase Inbound Connections
## Allows for +1M more FDs
## An FD is an integer value used as a traffic I/O pointer 
## on a connection with a Client.  
## The FD Int value is used to traffic packets between 
## User and Kernel Space.
fs.file-max = 1048576
```

How does the FRINX UniConfig distribution use NETCONF?
------------------------------------------------------

FRINX UniConfig uses a NETCONF southbound connectors to communicate with
downstream NETCONF-enabled devices. There are three options:

-   **uniconfig-native** - Using of raw device models for interaction
    with devices and still using Uniconfig transactions.
-   **translation units** - Translation units that map OpenConfig models
    to device models and vice-versa can be used for configuration of
    NETCONF devices using OpenConfig models.
-   **direct using of Netconf mount-points** - Modification of data
    under NETCONF mount-point but without option to use Uniconfig RPCs
    (data is exposed under 'yang-ext:mount' container).

UniConfig-native NETCONF
------------------------

UniConfig Native allows to communicate with network devices using their
native YANG data models (e.g.: Cisco YANG models, JunOS YANG models,
Calix YANG models, CableLabs YANG models, SROS YANG models, ...) to
manage configurations. With UniConfig Native is possible to mount
devices through NETCONF, sync configurations in their native format and
manage those devices without the need to develop translation units. Here
are some examples of NETCONF Native installation.

Examples
--------

[!ref](iosxr.md)
[!ref](junos.md)
[!ref](calix.md)
[!ref](sros.md)
[!ref](ocnos.md)
