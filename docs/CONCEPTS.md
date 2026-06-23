# Understanding the Lab — Segment Routing Concepts

A study companion to the lab. The README shows *how* it was built; this explains
*why* each piece works, so the theory sits next to the hands-on.

---

## The one idea

**Segment Routing is source routing.** The router at the *edge* decides the path and
writes it onto the packet as an ordered list of **segments** — each segment is an
instruction. The routers in the *middle* hold no per-path state; they just execute
whatever segment is on top and pass it on. In SR-MPLS a segment is an MPLS label; in
SRv6 it's an IPv6 address. Everything below is this one mechanism, used differently.

## Two kinds of segment

- **Prefix-SID (node-SID)** — "reach this router via the IGP shortest path." *Global*:
  the whole domain agrees (e.g. 16004 = R4). Builds shortest-path transport.
- **Adjacency-SID** — "go out this specific link." *Local* to one router; overrides
  the shortest path. Lets you pin exact hops.

A **segment list** is just a recipe mixing the two: "shortest-path to R3, then out
this link, then shortest-path to R4."

---

## Phase by phase — the why

**Phase 1 — IGP foundation.** IS-IS gives every router the topology map and shortest
paths. Prefix-SIDs ride the IGP shortest path, so the IGP must be solid first.

**Phase 2 — SR-MPLS.** The IGP itself advertises one label per loopback — the
prefix-SID. "16004 = R4" everywhere, network-wide. One fewer protocol layer.

**Phase 3 — TI-LFA.** Because SR can encode any path as a segment list, the router
pre-computes a loop-free backup for every destination and installs it before anything
breaks. Fast reroute (<50 ms) becomes universal and automatic.

**Phase 4 — SR-TE.** The headend encodes a deliberate segment list to force a
non-IGP path. The core holds *no* per-path state — all steering lives in the stack
the edge imposed. Traffic engineering without RSVP-TE's state explosion.

**Phase 5 — L3VPN.** Two label layers — an outer *transport* label (SR prefix-SID)
crosses the core, an inner *VPN* label tells the egress PE which VRF. The core is
oblivious to customers; the edge holds customer state.

**Phase 6 — SRv6.** A segment is now a 128-bit IPv6 address — a *locator* (which
node) plus a *function* (what to do). One IPv6 SID does both jobs the two MPLS labels
did — locator = transport, function = service.

**Phase 7 — EVPN-VPWS (L2VPN).** EVPN (BGP `l2vpn evpn` AF) signals a
point-to-point pseudowire. Over SRv6 the egress uses a `uDX2` SID — the L2 sibling
of `uDT4`. CE1 and CE2 learn each other's MAC, proving it's genuinely Layer 2.

---

## Key terms

| Term | Meaning |
|---|---|
| **SID** | Segment Identifier — one instruction (label in MPLS, IPv6 address in SRv6) |
| **Prefix-SID / Node-SID** | Global SID: "reach this node via shortest path" |
| **Adjacency-SID** | Local SID: "use this specific link" |
| **SRGB** | SR Global Block — label range for prefix-SIDs (default 16000–23999) |
| **TI-LFA** | Pre-computed loop-free backup path for sub-50ms reroute |
| **SR-TE / SR Policy** | An engineered path expressed as a segment list at the headend |
| **uN / uA / uDT4 / uDX2** | SRv6 uSID forms: node / adjacency / L3VPN / L2VPN service SIDs |
| **EVPN** | BGP control plane for L2 VPNs; carries MAC/Ethernet routes |
| **VPWS (E-Line)** | Point-to-point L2 pseudowire |
| **uSID** | Micro-SID — compression packing several SIDs into one IPv6 address |
