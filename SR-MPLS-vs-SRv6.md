# Choosing the L3VPN Transport: SR-MPLS vs SRv6

In this lab the *same* L3VPN (VRF `CUST-A`, CE1 ↔ CE2) was carried over **both**
SR-MPLS and SRv6. The service never changed — only the transport underneath it. So
the real question for a network designer is: **which transport do you assign the VPN
to, and why?** This guide lays out the trade-offs.

> The short version: **SR-MPLS** fits existing MPLS networks and constrained
> hardware; **SRv6** fits IPv6-native, greenfield, and multi-domain designs (notably
> 5G transport). They can also coexist and be migrated gradually.

---

## How they differ (the basis for the decision)

| Aspect | SR-MPLS | SRv6 |
|---|---|---|
| Segment is a… | MPLS label (4 bytes) | IPv6 address / SID (16 bytes) |
| Data plane | MPLS | Native IPv6 |
| Underlay needed | Every node forwards MPLS | IPv6 reachability; SID nodes need SRv6 support |
| Service encoding | Transport label + VPN label (two labels) | One SID (locator = transport, function = service, e.g. uDT4) |
| Header overhead | Low (4 bytes/label) | Higher (IPv6 + SRH), but **uSID** compresses it |
| Hardware maturity | Universal — runs on almost all routers | Needs SRv6-capable ASICs (especially for uSID) |
| Multi-domain reach | Needs MPLS end to end (or interworking) | Rides any IPv6 path, even non-MPLS segments |
| Operational maturity | Very mature, large skills base | Newer; growing fast, smaller skills base |

---

## When to choose SR-MPLS

- **Brownfield MPLS network.** Most operator cores already run MPLS. SR-MPLS reuses
  the existing data plane — you simply replace LDP/RSVP with IGP-distributed
  prefix-SIDs. Lowest-risk migration.
- **Older / mixed hardware.** Virtually every deployed router can forward MPLS.
  If some platforms can't do SRv6 (or uSID), SR-MPLS is the common denominator.
- **MTU-sensitive paths.** MPLS labels add only 4 bytes each; SRv6 adds a full IPv6
  header (and an SRH for multi-SID paths). Where MTU headroom is tight, MPLS is
  leaner.
- **You want the largest pool of experienced staff and tooling.** MPLS L3VPN is
  decades-proven and widely understood.

## When to choose SRv6

- **Greenfield or IPv6-native networks**, where you want to collapse the stack to a
  single IPv6 data plane and drop MPLS entirely — fewer protocols, simpler core.
- **5G / mobile transport.** SRv6 (with uSID) is the direction much of the mobile
  industry is taking; it pairs naturally with an IPv6-native 5G core.
- **Multi-domain / mixed underlays.** SRv6 packets are just IPv6, so they can cross
  any IPv6-capable network — datacenter, WAN, even segments that don't run MPLS —
  without MPLS interworking gateways.
- **Network programming / service insertion.** SRv6 functions (End.*, and service
  SIDs like uDT4) make the network programmable — useful for service chaining and
  richer per-flow behavior beyond what a flat MPLS label expresses.

---

## Quick decision checklist

Ask, in order:

1. **Is the core already MPLS?** Yes → SR-MPLS is the low-risk default.
2. **Does all relevant hardware support SRv6 (uSID)?** No → SR-MPLS.
3. **Is this new build / 5G transport / IPv6-native?** Yes → SRv6 is the strategic fit.
4. **Do you need to cross non-MPLS / multiple domains?** Yes → SRv6.
5. **Is MTU very tight and paths deep?** Leans SR-MPLS (or SRv6 with uSID).

If several answers conflict, that's normal — which is why coexistence exists.

---

## Coexistence and migration

You don't have to pick one forever. A common real-world path:

- Keep the existing **SR-MPLS** L3VPNs running on the current core.
- Stand up **SRv6** on new or upgraded nodes (as this lab did — both ran side by side
  on the same routers).
- Migrate services VRF by VRF, using interworking gateways where an MPLS island meets
  an SRv6 island.

The key enabler is the lesson from Phase 5 → Phase 6: the **service is independent of
the transport**. The VRF, RD/RT, and PE–CE setup don't change — only the encapsulation
does — so a VPN can be moved from MPLS to SRv6 without the customer noticing.

---

## Applied to a mobile operator

For a 5G-era mobile operator, the honest answer is usually **both, in sequence**: the
existing transport is almost certainly MPLS today (so SR-MPLS now), while new 5G
transport and longer-term simplification point toward **SRv6**. Being able to design,
run, and migrate between the two — which is what this lab demonstrates — is more
valuable than treating either as the single "right" answer.
