# TFTP-Assisted UDP NAT Hole Punching

## Abstract

This document describes an extension to the UDP NAT Hole Punching method for NAT traversal, as outlined in [RFC5128](https://www.rfc-editor.org/rfc/rfc5128.html).
On certain NAT devices, the [TFTP protocol](https://www.rfc-editor.org/rfc/rfc1350) Application Level Gateway (ALG) can be leveraged to significantly increase the success rate of establishing connections between peers.

## TFTP Application Level Gateway

Due to the simplicity of the [TFTP protocol](https://www.rfc-editor.org/rfc/rfc1350), ALGs implemented in NAT devices are typically not full TFTP protocol proxies but act merely as helpers that modify NAT translation behavior.

NAT devices that support the TFTP protocol will accept UDP responses to TFTP requests from *any* source port on the target IP address.

## NAT Types

As described in [RFC5128](https://www.rfc-editor.org/rfc/rfc5128.html), there are different types of NAT behaviors. Two are of particular interest:

- **Endpoint-Independent Mapping (EIM-NAT):** The same external address and port are used when a client sends UDP packets from the same internal port to multiple different endpoints.
- **Non-Endpoint-Independent Mapping (non-EIM-NAT):** The external port[^1] may vary when the client sends UDP packets to different endpoints, even when using the same internal port.

## Classic UDP Hole Punching

The UDP Hole Punching technique, as described in [RFC5128 §3.3](https://www.rfc-editor.org/rfc/rfc5128.html#section-3.3),
relies on both NAT devices being of the EIM-NAT type.

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

### Key Observations

Sending initial packets with a limited Time-To-Live (TTL) prevents the opposite NAT device from adding erroneous translation table entries—this can happen on some devices when receiving UDP packets for which no mapping exists yet.

Once a UDP packet stream is established, keep-alive packets must be sent periodically to prevent NAT devices from expiring the translation table entries.

## Failed UDP Hole Punching

Consider a scenario where one peer is behind an EIM-NAT and the other behind a non-EIM-NAT device, which assigns a random external port per endpoint.

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

### Key Observations

Client A may still attempt to establish a direct UDP packet stream by guessing the external port of Client B. However:

- Sending too many guesses quickly can stress NAT B (CPU load, translation table overflows, bandwidth).
- Sending guesses too slowly risks expiration of the required translation table entries.

A useful technique to reduce this impact, based on [the birthday paradox](https://en.wikipedia.org/wiki/Birthday_problem) is described in [How NAT traversal works](https://tailscale.com/blog/how-nat-traversal-works) on Tailscale blog.

## TFTP-Assisted UDP Hole Punching

Now consider the case where NAT A supports TFTP ALG and is EIM-NAT, while NAT B is non-EIM-NAT. The well-known port for TFTP protocol is UDP/69.

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

### Key Observations

Sending a single packet to port 69 on the peer's external IP address allows NAT A to create a more permissive translation table entry due to the TFTP ALG behavior.

This method can be combined with the classic approach by sending an additional UDP packet to port 69, ideally after a short timeout if standard hole punching fails.

## Applications

The utility of this method depends largely on the prevalence of NAT devices with TFTP ALG enabled. Some data suggests that certain DSL routers shipped by ISPs include this feature enabled by default.
Further data collection is needed to assess the general viability of this technique for enhancing peer-to-peer connectivity.

However, due to the low complexity and minimal cost associated with the described method, it could still be useful even if only a minority of residential routers—e.g., one in several—have TFTP ALG enabled.

---

### Metadata

```yaml
title: TFTP-Assisted UDP NAT Hole Punching
author: Rafał Kupka
email: r.kupson@gmail.com
date: 2025-06-09
```

[^1]: External IP address can vary too but it's less common.
