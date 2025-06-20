# UDP Hole Punching

Exploring possible improvements to the [RFC5128 UDP Hole Punching](https://www.rfc-editor.org/rfc/rfc5128.html#section-3.3) technique.

## Contents

- [TFTP-NAT-UHP.md](/TFTP-NAT-UHP.md) - helping EIM-NAT (with TFTP-ALG) &lt;-&gt; non-EIM-NAT scenarios
- [TFTP-NAT-GUESS.md](/TFTP-NAT-GUESS.md) - enabling non-EIM-NAT (with TFTP-ALG) &lt;-&gt; non-EIM-NAT scenarios without a relay server

## Tools

- [bin/tftptest](/bin/tftptest) - Simple test tool for checking TFTP connectivity
- [bin/tftpspray](/bin/tftpspray) - Helper tool for the TFTP-Assisted [birthday paradox](https://en.wikipedia.org/wiki/Birthday_problem) port guessing

# How to Help

You can contribute by sharing information about your ISP and router in [issue #1](https://github.com/kupson/udp-hole-punching/issues/1).

#### TODO

- Investigate SIP-ALG.
