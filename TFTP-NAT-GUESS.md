# TFTP-Assisted port guessing for UDP NAT Hole Punching

## Abstract

This document extends the previously described [TFTP-Assisted UDP NAT Hole Punching](/TFTP-NAT-UHP.md) mechanism to work when both NAT devices are non-EIM-NAT. If one of the sides supports TFTP-ALG, it's possible to establish a direct UDP packet stream between devices behind those NATs.

## The Hard Case

The classic UDP Hole Punching as described in [RFC5128 §3.3](https://www.rfc-editor.org/rfc/rfc5128.html#section-3.3) works well if both devices use EIM-NAT.

If only one side has endpoint-independent mappings, establishing a bidirectional UDP packet stream is possible but requires guessing the opposite side port number (~64,000 ports). [The birthday paradox](https://en.wikipedia.org/wiki/Birthday_problem) can help significantly, as described in [How NAT traversal works](https://tailscale.com/blog/how-nat-traversal-works) on Tailscale blog.

For the case when both sides are non-EIM-NAT devices -- only a relay server can help as guessing two port numbers simultaneously would be prohibitively costly.

## The Hard Case plus TFTP-ALG

Having a NAT device with TFTP-ALG support on one side greatly reduces the number of combinations to guess, as it allows that side to accept any source port from the other side. That makes the situation similar in complexity to the EIM-NAT &lt;-&gt; non-EIM-NAT case.

### Live Example

One side supports TFTP-ALG but has a non-EIM-NAT Carrier-Grade NAT (O2 UK cellular data):

```
$ ./bin/tftptest
157.245.47.226:69 for 0.0.0.0:54406 returns address: 82.132.231.141:31688
207.154.212.211:69 for 0.0.0.0:54406 returns address: 82.132.231.141:44360
```

Let's check the IP address of the opposite side:

```
$ curl -4 ifconfig.co
90.199.252.188
```

The [`tftpspray`](/bin/tftpspray) tool opens 256 UDP ports and primes the translation table mappings on the non-EIM-NAT device (the one that supports TFTP-ALG)[^1]:

```
$ ./bin/tftpspray -l 90.199.252.188
Done sending TFTP GET requests.
```

The other side will send the packets. Due to the TFTP-ALG behaviour, the source port doesn't matter -- only the destination port needs to be guessed:

```
$ ./bin/tftpspray -s 82.132.231.141
```

Now let's look at the full command output from both sides:

```
$ ./bin/tftpspray -l 90.199.252.188
Done sending TFTP GET requests.
.
PING/PONG received from: ('90.199.252.188', 53747) on ('0.0.0.0', 33191)
Total packets received: 1
Total packets sent: 256
```

```
$ ./bin/tftpspray -s 82.132.231.141
.
PING/PONG received from: ('82.132.231.141', 40044) on ('0.0.0.0', 53747)

Total packets received: 1
Total packets sent: 869
```

Note that non-EIM-NAT devices can be freely chained (e.g., in a double-NAT setup), as long as all of them support TFTP-ALG.

---

### Metadata

```yaml
title: TFTP-Assisted port guessing for UDP NAT Hole Punching
author: Rafał Kupka
email: r.kupson@gmail.com
date: 2025-06-20
```

[^1]: By sending a TFTP GET request to port 69 on the destination IP.