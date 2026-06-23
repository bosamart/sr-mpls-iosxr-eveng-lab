# Segment Routing (SR-MPLS & SRv6) on Cisco IOS XRv9000 — EVE-NG Lab

A hands-on lab building a service-provider style core that forwards traffic using
**Segment Routing** — no LDP, no RSVP-TE. Progresses from plain IGP up to
traffic engineering and an L3VPN, then carries that same VPN over **both**
SR-MPLS and SRv6 data planes.

**GitHub:** https://github.com/bosamart/sr-mpls-iosxr-eveng-lab

## Lab Environment
| Component | Detail |
|---|---|
| Emulator | EVE-NG (Community/Pro) |
| Node image | Cisco IOS XRv9000 (24.3.1) |
| Core/PE nodes | 4 × XRv9000 (R1–R4) |
| CE nodes | 2 × Cisco IOS XRv9000 (CE1, CE2) |
| IGP | IS-IS Level-2-only |
| Transport | SR-MPLS (SRGB 16000–23999) + SRv6 (fcbb:bb00::/32) |

## Topology
```
             +-------+
    +--------|  R2   |--------+
    |        |  (P)  |        |
    |        +---+---+        |
    |            | cross-link |
+---+----+       |         +--+-----+
|  R1    |       |         |  R4    |
|  (PE)  |       |         |  (PE)  |
+---+----+       |         +--+-----+
    |        +---+---+        |
    +--------|  R3   |--------+
    |        |  (P)  |        |
    |        +-------+        |
 +--+--+                   +--+--+
 | CE1 |                   | CE2 |
 +-----+                   +-----+
```

## SID / Address Plan
| Node | Role | Loopback | Prefix-SID | SRv6 Locator |
|------|------|----------|------------|--------------|
| R1 | PE | 1.1.1.1/32 | 16001 | fcbb:bb00:1::/48 |
| R2 | P  | 2.2.2.2/32 | 16002 | fcbb:bb00:2::/48 |
| R3 | P  | 3.3.3.3/32 | 16003 | fcbb:bb00:3::/48 |
| R4 | PE | 4.4.4.4/32 | 16004 | fcbb:bb00:4::/48 |

## Phases Completed
- [x] Phase 1 — IS-IS baseline
- [x] Phase 2 — SR-MPLS (prefix-SIDs, no LDP)
- [x] Phase 3 — TI-LFA fast reroute (100% coverage)
- [x] Phase 4 — SR-TE explicit-path policy
- [x] Phase 5 — L3VPN over SR-MPLS
- [x] Phase 6 — L3VPN over SRv6 (uDT4)
- [x] Phase 7 — EVPN-VPWS L2VPN over SR-MPLS and SRv6 (uDX2)
