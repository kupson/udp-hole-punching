# TFTP assisted UDP NAT Hole Punching

## Abstract

This document describes an extension to the UDP NAT Hole Punching method of NAT Traversal described in [RFC5128](https://www.rfc-editor.org/rfc/rfc5128.html).
On some NAT devices the [TFTP protocol](https://www.rfc-editor.org/rfc/rfc1350) Application Level Gateway (ALG) can be used to greatly increase ability to connect to remote peers.

## TFTP Application Level Gateway

Due to simplicity of the [TFTP protocol](https://www.rfc-editor.org/rfc/rfc1350) the ALG in devices performing NAT is typically not a full TFTP protocol proxy but merely a helper to the connection tracking code.

NAT devices compatible with TFTP protocol do accept UDP packet reply to TFTP request from *any* port of the target IP.

## NAT types

The [RFC5128](https://www.rfc-editor.org/rfc/rfc5128.html) document describes different NAT types. Two important for us will be:

- Endpoint-Independent Mapping type (EIM-NAT) - The same external address and port is used when given client port is used to send UDP packets to multiple different endpoints.
- Non Endpoint-Independent Mapping type (non-EIM-NAT) - The external address may stay the same but the external port will be different when the same client port will be used to send UDP packets to different endpoints.

## Classic UDP Hole Punching

The UDP Hole Punching NAT Traversal method as described in 
[RFC5128 3.3.](https://www.rfc-editor.org/rfc/rfc5128.html#section-3.3)
requires both-end NAT Devices to be Endpoint-Independent Mapping type (EIM-NAT).

```mermaid
sequenceDiagram
    participant ClientA as Client A<br />192.168.1.33
    participant NatA as NAT Device A<br />198.51.100.1
    participant S as Rendezvous Server S
    participant NatB as NAT Device B<br />203.0.113.1
    participant ClientB as Client B<br />10.0.0.44

ClientA ->> NatA: from 192.168.1.33:33333 to S
NatA ->> S: from 198.51.100.1:33333 to S
ClientB ->> NatB: from 10.0.0.44:44444 to S
NatB ->> S: from 203.0.113.1:44444 to S
Note over S: Randezvous server now knows external addresses and ports assigned by NAT Devices A and B
S -->> ClientA: Client B is on 203.0.113.1:44444
S -->> ClientB: Client A is on 198.51.100.1:33333
Note over ClientA,ClientB: Trying to establish a direct UDP packet stream
ClientA ->> NatA: from 192.168.1.33:33333 to 203.0.113.1:44444
NatA ->> NatB: from 198.51.100.1:33333 to 203.0.113.1:44444
Note over NatA: translation table entry added
Note over NatB: initial packets can be sent with limited TTL<br /> and don't even reach Nat Device B
ClientB ->> NatB: from 10.0.0.44:44444 to 198.51.100.1:33333
NatB ->> NatA: from 203.0.113.1:44444 to 198.51.100.1:33333
Note over NatB: translation table entry added
Note over NatA: initial packets can be sent with limited TTL<br /> and don't even reach Nat Device A
Note over ClientA,ClientB: both NAT Devices should have correct mapping now
ClientA ->> NatA: from 192.168.1.33:33333 to 203.0.113.1:44444
NatA ->> NatB: from 198.51.100.1:33333 to 203.0.113.1:44444
NatB ->> ClientB: from 198.51.100.1:33333 to 10.0.0.44:44444
ClientB ->> NatB: from 10.0.0.44:44444 to 198.51.100.1:33333
NatB ->> NatA: from 203.0.113.1:44444 to 198.51.100.1:33333
NatA ->> ClientA: from 203.0.113.1:44444 to 192.168.1.33:33333
Note over ClientA,ClientB: UDP packet stream established
```

Sending initial packets with limited Time-To-Live prevents the opposite NAT device from adding erroreus translation table entries.
This can happen on some devices when they receive a UDP packet without existing translation table entry.

Once established the UDP packet stream would need to kept alive by sending keep-alive packets. That should prevent NAT Devices from removing the needed transation table entries.

## Failed UDP Hole Punching

Let's examine the scenario where one of the NAT devices is still a EIM-NAT type but the peer is a non-EIM-NAT type device that pick random port for each endpoint.

```mermaid
sequenceDiagram
    participant ClientA as Client A<br />192.168.1.33
    participant NatA as NAT Device A (EIM-NAT)<br />198.51.100.1
    participant S as Rendezvous Server S
    participant NatB as NAT Device B (non-EIM-NAT)<br />203.0.113.1
    participant ClientB as Client B<br />10.0.0.44

ClientA ->> NatA: from 192.168.1.33:33333 to S
NatA ->> S: from 198.51.100.1:33333 to S
ClientB ->> NatB: from 10.0.0.44:44444 to S
NatB ->> S: from 203.0.113.1:36890 to S
Note over S: Randezvous server now knows external addresses and ports assigned by NAT Devices A and B
S -->> ClientA: Client B is on 203.0.113.1:36890<br />(true only for server S)
S -->> ClientB: Client A is on 198.51.100.1:33333
Note over ClientA,ClientB: Trying to establish a direct UDP packet stream
ClientA ->> NatA: from 192.168.1.33:33333 to 203.0.113.1:36890
NatA ->> NatB: from 198.51.100.1:33333 to 203.0.113.1:36890
Note over NatA: translation table entry added<br />(incorrect)
Note over NatB: initial packets can be sent with limited TTL<br /> and don't even reach Nat Device B
ClientB ->> NatB: from 10.0.0.44:44444 to 198.51.100.1:33333
NatB ->> NatA: from 203.0.113.1:48531 to 198.51.100.1:33333
Note over NatB: translation table entry added<br />(incorrect)
Note over NatA: initial packets can be sent with limited TTL<br /> and don't even reach Nat Device A
Note over ClientA,ClientB: both NAT Devices have mapping now, but it's incorrect
ClientA ->> NatA: from 192.168.1.33:33333 to 203.0.113.1:36890
NatA ->> NatB: from 198.51.100.1:33333 to 203.0.113.1:36890
Note over NatB: destination port mismatch<br />36890 != 48531<br />dropped
ClientB ->> NatB: from 10.0.0.44:44444 to 198.51.100.1:33333
NatB ->> NatA: from 203.0.113.1:48531 to 198.51.100.1:33333
Note over NatA: source port mismatch<br />36890 != 48531<br />dropped
Note over ClientA,ClientB: packets dropped on both NAT devices
```

In this case the Client A can still try to establish a direct UDP stream to Client B by guessing correct port number in translation table on NAT Device B.

This will take time and will require maybe thausends of packets to be sent to NAT Device B.
If Client A will send those packets quickly that may stress NAT device B - overflow of translation table, higher CPU usage due to flood of UDP packets, bandwidtch exhaustion could be a problem.
Sending those guessing packets slowly can cause the correct translation table entry (on NAT B) to be removed due to timeout before Client A would be able to guess it.

## TFTP assisted UDP Hole Punching

Let's see how introducing TFTP ALG on NAT Device A can change the situation. The well-known port for TFTP protocol is UDP/69.
The NAT device A is a EIM-NAT type with TFTP ALG (connection tracking helper). The NAT device B is a non-EIM-NAT type device that pick random port for each endpoint.

```mermaid
sequenceDiagram
    participant ClientA as Client A<br />192.168.1.33
    participant NatA as NAT Device A (EIM-NAT)<br />198.51.100.1
    participant S as Rendezvous Server S
    participant NatB as NAT Device B (non-EIM-NAT)<br />203.0.113.1
    participant ClientB as Client B<br />10.0.0.44

ClientA ->> NatA: from 192.168.1.33:33333 to S
NatA ->> S: from 198.51.100.1:33333 to S
ClientB ->> NatB: from 10.0.0.44:44444 to S
NatB ->> S: from 203.0.113.1:51021 to S
Note over S: Randezvous server now knows external addresses and ports assigned by NAT Devices A and B
S -->> ClientA: Client B is on 203.0.113.1:51021<br />(true only for server S)
S -->> ClientB: Client A is on 198.51.100.1:33333
Note over ClientA,ClientB: Trying to establish a direct UDP packet stream
ClientA ->> NatA: from 192.168.1.33:33333 to 203.0.113.1:69
NatA ->> NatB: from 198.51.100.1:33333 to 203.0.113.1:69
Note over NatA: TFTP translation table entry added<br />will accept reply from any port on 203.0.113.1
Note over NatB: initial packets will be dropped<br />either due to a TTL limit or because no service should listen on 203.0.113.1:69
ClientB ->> NatB: from 10.0.0.44:44444 to 198.51.100.1:33333
NatB ->> NatA: from 203.0.113.1:36747 to 198.51.100.1:33333
Note over NatB: translation table entry added
Note over NatA: initial packets can be sent with limited TTL<br /> and don't even reach Nat Device A
Note over ClientA,ClientB: both NAT Devices have mapping now
ClientB ->> NatB: from 10.0.0.44:44444 to 198.51.100.1:33333
NatB ->> NatA: from 203.0.113.1:44576 to 198.51.100.1:33333
Note over NatA: translation table entry updated
NatA ->> ClientA: from 203.0.113.1:44576 to 192.168.1.33:33333
Note over ClientA: now Client A knows the external port of client B
ClientA ->> NatA: from 192.168.1.33:33333 to 203.0.113.1:44576
NatA ->> NatB: from 198.51.100.1:33333 to 203.0.113.1:44576
NatB ->> ClientB: from 198.51.100.1:33333 to 10.0.0.44:44444
Note over ClientA,ClientB: UDP packet stream established
```

This method can be combined with the classic approach from RFC5128 by sending one additional UDP packet to port 69 of the opposite peer external IP address. Possibly after a short timeout period.

## Applications

The applicability of the described mechanism depends a lot on the question how wirespread are the NAT devices with TFTP ALG support enabled. Some data may suggest that at least some ISPs ship DSL routers with TFTP ALG enabled by default.
It would be nice to gather more examples and estimate better how much this method would help to create better Peer-to-peer connections.

#### Metadata

---
title: TFTP assisted UDP NAT Hole Punching
author: Rafał Kupka
e-mail: r.kupson@gmail.com
date: 2025-06-09
---

