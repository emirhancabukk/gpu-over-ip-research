# RTT Survey: Home Connection to AWS Regions (2026-08-31)

Measured from the same residential connection in Turkey that the phase 4 runs used, to
decide whether the ~66 ms round trip to Frankfurt is mostly physics or mostly routing.

## Method

- **TCP connect time** to `ec2.<region>.amazonaws.com:443`, measured in Python: the name
  is resolved first, then only `socket.connect()` is timed with `time.perf_counter()`.
  7 samples per region, 0.15 s apart, 5 s timeout. This excludes the DNS lookup that
  curl's `time_connect` includes (see `phase0-network-measurement.md`).
- **ICMP** was tried against the two fastest endpoints (`ping -c 15`): 15/15 packets
  lost on both. ICMP is filtered on this path, so TCP connect time is the only usable
  RTT measurement here.
- **Physical floor:** great-circle distance from Istanbul (used as the reference point)
  to a city in the region, travelled twice at 5 µs/km, the speed of light in fiber.
  Real cable routes are longer than great circles, so the floor is optimistic.

## Results

| Region | Reference city | Median | Min | Max | Distance | Floor | Median / floor |
|---|---|---|---|---|---|---|---|
| `eu-south-1` | Milan | **56.5 ms** | 55.1 | 59.9 | 1,673 km | 16.7 ms | 3.4× |
| `eu-central-1` | Frankfurt | 66.1 ms | 64.2 | 67.5 | 1,867 km | 18.7 ms | 3.5× |
| `eu-south-2` | Zaragoza | 91.1 ms | 84.6 | 178.4 | 2,482 km | 24.8 ms | 3.7× |
| `eu-central-2` | Zurich | 91.2 ms | 84.8 | 179.9 | 1,770 km | 17.7 ms | 5.2× |
| `eu-west-1` | Dublin | 92.5 ms | 88.3 | 171.8 | 2,955 km | 29.5 ms | 3.1× |
| `me-central-1` | Dubai | 160.3 ms | 157.8 | 164.8 | 2,995 km | 29.9 ms | 5.4× |
| `me-south-1` | Manama | — | — | — | 2,576 km | 25.8 ms | no connection on any sample |

## Reading

- The two nearest regions sit at 3.4–3.5× their physical floor. Three to four times the
  great-circle floor is common for routed commercial internet, so the Frankfurt path is
  ordinary, not badly routed.
- Moving from Frankfurt to Milan would save about 10 ms (15%) per round trip. Whether
  `g5` GPU instances are offered in `eu-south-1` was not checked.
- Zurich and the UAE are the outliers (5.2× and 5.4×): their traffic evidently takes a
  longer route than the distance suggests.
- Conclusion for the project: a better region buys at most ~15%. Reducing the
  **number** of round trips per Vulkan call is the only lever with large headroom.

## Compared with phase 0

Phase 0 measured Milan at 128 ms on 27 August, with curl; this survey measured 56.5 ms
four days later. Phase 0's number includes the DNS lookup and was a single request, which
may explain part of the difference; it was not investigated further. Frankfurt agrees
between the two (64 ms and 66.1 ms).
