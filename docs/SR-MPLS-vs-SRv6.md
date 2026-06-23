# Choosing the L3VPN Transport: SR-MPLS vs SRv6

The same L3VPN (VRF `CUST-A`) was carried over both SR-MPLS and SRv6 in this lab.
The service never changed — only the transport underneath it.

> **Short version:** SR-MPLS fits existing MPLS networks and constrained hardware;
> SRv6 fits IPv6-native, greenfield, and multi-domain designs (notably 5G transport).

---

## How they differ

| Aspect | SR-MPLS | SRv6 |
|---|---|---|
| Segment is a… | MPLS label (4 bytes) | IPv6 address / SID (16 bytes) |
| Data plane | MPLS | Native IPv6 |
| Service encoding | Transport label + VPN label | One SID (locator + function) |
| Header overhead | Low (4 bytes/label) | Higher, but uSID compresses it |
| Hardware maturity | Universal | Needs SRv6-capable ASICs |
| Multi-domain reach | Needs MPLS end to end | Rides any IPv6 path |
| Operational maturity | Very mature | Newer, growing fast |

---

## When to choose SR-MPLS

- Brownfield MPLS network — reuses existing data plane, lowest-risk migration
- Older / mixed hardware — virtually every router can forward MPLS
- MTU-sensitive paths — MPLS labels add only 4 bytes each
- Largest pool of experienced staff and tooling

## When to choose SRv6

- Greenfield or IPv6-native networks — collapse to single IPv6 data plane
- 5G / mobile transport — SRv6 with uSID is the industry direction
- Multi-domain / mixed underlays — SRv6 crosses any IPv6 network
- Network programming / service insertion

---

## Quick decision checklist

1. Is the core already MPLS? → SR-MPLS (low-risk default)
2. Does all hardware support SRv6 (uSID)? No → SR-MPLS
3. New build / 5G transport / IPv6-native? → SRv6
4. Need to cross non-MPLS / multiple domains? → SRv6
5. MTU very tight? → SR-MPLS (or SRv6 with uSID)

---

## Coexistence and migration

Both ran side-by-side on the same routers in this lab. Real-world path:
- Keep existing SR-MPLS L3VPNs running
- Stand up SRv6 on new/upgraded nodes
- Migrate services VRF by VRF

The key: **the service is independent of the transport**. VRF, RD/RT, and PE–CE
setup don't change — only the encapsulation does. A VPN can move from MPLS to SRv6
without the customer noticing.

---

## Applied to a mobile operator (Cellcard context)

For a 5G-era mobile operator: existing transport is almost certainly MPLS today
(SR-MPLS now), while new 5G transport and longer-term simplification point toward
SRv6. Being able to design, run, and migrate between both is what this lab demonstrates.
