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

**Phase 8 — SR-PCE + ODN.** Instead of the headend hand-building the segment list
(Phase 4), a central **controller (SR-PCE)** learns the whole topology — fed to it by
**BGP-LS** — and computes paths on request. A service route tagged with a **color**
(an intent number) makes the headend ask the PCE, over **PCEP**, for a matching path,
which it then installs automatically (**ODN** = On-Demand Next-hop). Intent in, path out,
no per-destination config. This is the controller-driven model behind 5G transport.

---

## Concepts the lab leans on (common trip-ups)

- **PHP (penultimate-hop popping).** The *second-to-last* router removes the transport
  label so the egress receives a clean packet. That's why a directly-connected
  prefix-SID shows `Pop` in `show mpls forwarding`, while a far one shows a real label.
- **Index vs label.** You configure a prefix-SID **index**; the label = **SRGB base
  (16000) + index**. Index 4 → label 16004. The *index* is the portable identity — if
  the SRGB base ever changes, the index stays and the label just shifts.
- **RD vs RT.** The **Route Distinguisher** makes prefixes *unique* (so two customers
  can reuse the same IP range); the **Route Target** controls *membership* (who imports
  the route). Different jobs — that's the whole point of the CUST-B demo VRF.
- **Why the R2–R3 cross-link matters.** It provides a loop-free alternate. *That's* what
  lets TI-LFA build a backup (Phase 3) and gives SR-TE a meaningful non-shortest path
  (Phase 4). Without it, the diamond is just two parallel paths and neither is interesting.
- **"Nexthop in vrf default."** A VPN route resolves its BGP next-hop in the *global*
  table — the transport label gets the packet to the egress PE, the VPN label/SID picks
  the VRF. This separation is why the core never holds customer routes.

---

## Can I explain it? — self-test

Don't peek. If you can answer these out loud in your own words, you understand the lab.
(Answers are throughout the README, the verify logs, and the sections above.)

1. Why is `metric-style wide` mandatory before SR will work?
2. What's the difference between a prefix-SID and an adjacency-SID, and when would you
   need each?
3. In a Phase 4 traceroute the label stack shrinks by one SID per hop — why?
4. Why does TI-LFA on this topology show `Local-LFA` instead of building a TI-LFA tunnel?
5. eBGP session is up but zero prefixes arrive — what's the IOS XR cause and fix?
6. In L3VPN, what are the *two* labels on a customer packet and what does each do?
7. `uDT4` vs `uDX2` — what does each instruct the egress PE to do?
8. How does one SRv6 SID replace the two MPLS labels of an L3VPN?
9. In Phase 8, what exactly *triggers* ODN to build a policy, and who computes the path?
10. Why did forcing a non-shortest ODN path fail on this XRv9000 build?

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
| **PHP** | Penultimate-hop popping — 2nd-to-last router strips the transport label |
| **RD / RT** | Route Distinguisher = prefix uniqueness; Route Target = VPN membership |
| **Locator / Function** | SRv6 SID parts: which node to reach / what it does on arrival |
| **Color** | A number on a BGP route encoding intent (e.g. low-latency) for steering |
| **SR-PCE** | Path Computation Element — controller that computes SR-TE paths on request |
| **PCEP / PCC** | Protocol between headend (PCC) and the PCE |
| **BGP-LS** | Carries the IGP topology to a controller (feeds the PCE) |
| **ODN** | On-Demand Next-hop — auto-create an SR-TE policy when a colored route appears |
