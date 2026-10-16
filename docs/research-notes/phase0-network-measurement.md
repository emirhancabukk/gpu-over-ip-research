# Phase 0 - Network Measurement Notes (2026-08-27)

> **Later note (2026-10-16):** curl's `time_connect` is measured from the start of the
> request, so it includes the DNS lookup. The figures below are upper bounds on the
> RTT, not the RTT itself. A later survey that times only the TCP handshake is in
> [`rtt-survey-2026-08-31.md`](rtt-survey-2026-08-31.md).

## Method
ICMP (ping/traceroute) was blocked or did not work in this environment (`ping` lost 5/5 packets, `traceroute` is not installed).
Instead, the TCP connect + TLS handshake time was measured with `curl` (HTTPS requests to the AWS EC2 API endpoints).
`time_connect` roughly corresponds to 1 RTT (SYN → SYN/ACK).

## Results (AWS EU regions, TCP connect time)

| Region | Endpoint | TCP connect | TLS done | Total |
|---|---|---|---|---|
| eu-central-1 (Frankfurt) | ec2.eu-central-1.amazonaws.com | 64ms | 238ms | 427ms |
| eu-west-1 (Ireland) | ec2.eu-west-1.amazonaws.com | 127ms | 317ms | 513ms |
| eu-west-2 (London) | ec2.eu-west-2.amazonaws.com | 151ms | 324ms | 511ms |
| eu-south-1 (Milan) | ec2.eu-south-1.amazonaws.com | 128ms | 304ms | 491ms |

**Lowest latency: Frankfurt (eu-central-1), ~64ms TCP connect.**

## Important caveat / to be verified

This measurement was made through Claude Code's Bash tool, so it is **not certain**
whether it goes over the real ISP path of the user's gaming/desktop machine or over a
different network path. The 64ms figure is higher than expected for good fiber peering
(Istanbul–Frankfurt is usually in the 25–35ms band). This may reflect (a) the real ISP
routing quality, (b) first-connection (TCP+TLS) overhead beyond the pure RTT, or (c) this
tool measuring over a different network path.

**To do:** repeat the same measurement from the user's real gaming computer, ideally with
raw ICMP ping (`ping -c 20 <frankfurt-ip>`) or with `mtr`, and if possible bring up a real
AWS g5 instance and ping it directly (the instance's own IP, not the EC2 API endpoint: API
endpoint routing can differ from instance routing).

## Next step
There is no AWS account + CLI setup (`aws` command not found). Before opening a real GPU instance and pinging it:
1. Clarify whether the AWS account is ready and whether there is credit card / budget approval (g5.xlarge ~$1/hour, billed hourly if not shut down).
2. Install the AWS CLI and configure credentials.
3. Open a g5.xlarge in Frankfurt, measure both ICMP and TCP while it is up, then shut it down immediately.
