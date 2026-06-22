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

**Phase 1 — IGP foundation.** *Problem:* you need basic reachability before anything
else. *Mechanism:* IS-IS gives every router the topology map and shortest paths.
*Insight:* prefix-SIDs ride the IGP shortest path, so the IGP must be solid first.
The two equal-cost paths to R4 (ECMP) are just IS-IS on the diamond.

**Phase 2 — SR-MPLS.** *Problem:* classic MPLS needs a separate protocol (LDP/RSVP)
to hand out labels. *Mechanism:* the IGP itself advertises one label per loopback —
the prefix-SID. *Insight:* "16004 = R4" everywhere, network-wide. You removed an
entire protocol layer; this globally-agreed label is the foundation for everything
after.

**Phase 3 — TI-LFA.** *Problem:* link failures cause packet loss until the IGP
reconverges. *Mechanism:* because SR can encode any path as a segment list, the
router pre-computes a loop-free backup for every destination and installs it before
anything breaks. *Insight:* fast reroute (<50 ms) becomes universal and automatic.
Your backup showed *Local-LFA* because the cross-link gave a directly loop-free
neighbor — TI-LFA only builds a repair tunnel when no simple alternate exists.

**Phase 4 — SR-TE.** *Problem:* sometimes you need a path the IGP wouldn't choose
(latency, disjointness). *Mechanism:* the headend encodes a deliberate segment list
(`{16002, 16003, 16004}` = via R2, via R3, to R4). *Insight:* the core holds *no*
per-path state — all steering lives in the stack the edge imposed. Traffic
engineering without RSVP-TE's state explosion.

**Phase 5 — L3VPN.** *Problem:* carry many customers across one shared core. *Mechanism:*
two label layers — an outer *transport* label (SR prefix-SID) crosses the core, an
inner *VPN* label tells the egress PE which VRF. *Insight:* the core only reads the
outer label and is oblivious to customers; the edge holds customer state. That's why
it scales. The VRF route resolving `nexthop in vrf default` is this recursion onto
the global SR transport.

**Phase 6 — SRv6.** *Problem:* simplify and go IPv6-native (5G transport direction).
*Mechanism:* a segment is now a 128-bit IPv6 address — a *locator* (which node) plus a
*function* (what to do). The node-SID becomes a `uN` SID; the VPN label becomes a
`uDT4` service SID. *Insight:* one IPv6 SID does both jobs the two MPLS labels did —
locator = transport ("reach R4"), function = service ("decap + VRF lookup"). The core
just routes plain IPv6.

**Phase 7 — EVPN-VPWS (L2VPN).** *Problem:* operators also sell Layer 2 (E-Line)
services, not just L3. *Mechanism:* the CE port becomes an L2 attachment circuit;
EVPN (BGP `l2vpn evpn` AF, Route-Type 1) signals a point-to-point pseudowire across
the same SR core. Over SRv6 the egress uses a `uDX2` SID. *Insight:* `uDX2` is the L2
sibling of `uDT4` — same encapsulation machinery, but the egress instruction is
"decap and hand the frame to this port" instead of "decap and route." CE1 and CE2 sit
in one subnet and learn each other's MAC, proving it's genuinely Layer 2 across a
routed core. L2 and L3 services run side by side on one core.

---

## The synthesis

Every phase is the same act: **put the right segment list on the packet at the edge,
and let the core just follow the top segment.**

- Fast reroute = a *backup* segment list.
- Traffic engineering = a *chosen* segment list.
- L3VPN = a *service* segment riding a *transport* segment.
- SRv6 = all of the above, with segments that are IPv6 addresses doing double duty.

The core's job never changes: read the top segment, act, move on.

---

## Key terms

| Term | Meaning |
|---|---|
| **IGP / IS-IS** | Interior routing protocol; here it also distributes SR labels |
| **SID** | Segment Identifier — one instruction (a label in MPLS, an IPv6 address in SRv6) |
| **Prefix-SID / Node-SID** | Global SID: "reach this node via shortest path" |
| **Adjacency-SID** | Local SID: "use this specific link" |
| **SRGB** | SR Global Block — label range for prefix-SIDs (default 16000–23999); label = base + index |
| **PHP** | Penultimate-Hop Popping — the second-to-last router removes the label |
| **TI-LFA / LFA** | Pre-computed loop-free backup path for sub-50ms reroute |
| **SR-TE / SR Policy** | An engineered path expressed as a segment list at the headend |
| **Binding-SID** | One label that represents an entire SR policy/path |
| **Autoroute / ODN / Color** | Methods to steer traffic *into* an SR policy |
| **VRF / RD / RT** | Customer routing table / route distinguisher / route target (import-export control) |
| **MP-BGP VPNv4** | Carries customer (VPN) routes between PEs with a VPN label |
| **Locator (SRv6)** | IPv6 prefix per node; SIDs are allocated from it |
| **Function (SRv6)** | The instruction encoded in a SID (End, End.X, End.DT4 …) |
| **uN / uA / uDT4** | uSID forms of node-SID / adjacency-SID / "decap + IPv4 VRF lookup" |
| **EVPN** | BGP control plane for L2 (and L3) VPNs; carries MAC/Ethernet routes |
| **VPWS (E-Line)** | Point-to-point L2 pseudowire — two ports act as one wire |
| **uDX2 (End.DX2)** | SRv6 L2VPN service SID: "decap + hand the frame to this L2 port" |
| **AC (attachment circuit)** | The CE-facing L2 port (`l2transport`) bound to a pseudowire |
| **uSID** | Micro-SID — compression packing several SIDs into one IPv6 address |
| **H.Encaps.Red** | SRv6 headend encapsulation (ingress wraps the packet in IPv6 + SID) |

---

## Questions you should be able to answer (interview prep)

**Prefix-SID vs adjacency-SID?**
Prefix-SID is global ("reach this node via shortest path", e.g. 16004 = R4).
Adjacency-SID is local ("use this exact link"), overriding shortest path. Prefix-SIDs
build transport; adjacency-SIDs pin hops for TE.

**What is the SRGB?**
The label range reserved for prefix-SIDs (default 16000–23999). A node's label = SRGB
base + its index (index 4 → 16004). All nodes share the same SRGB so labels are
consistent domain-wide.

**Why does SR remove the need for LDP/RSVP?**
The IGP advertises the labels itself, so there's no separate label-distribution
protocol — one protocol does routing and labels together.

**How does TI-LFA guarantee a loop-free backup?**
Because SR can encode any path as a segment list, TI-LFA can steer the repair packet
along a guaranteed loop-free path (even where a simple neighbor would loop) and
pre-installs it for instant failover.

**Why did your backup show Local-LFA, not TI-LFA?**
A simple loop-free alternate already existed (the R2–R3 cross-link). TI-LFA only
builds an explicit repair tunnel when no simple LFA exists, so it correctly used the
cheaper plain LFA.

**What do the two labels in an MPLS L3VPN do?**
Outer = transport (SR prefix-SID) to cross the core. Inner = VPN label telling the
egress PE which VRF. The core reads only the outer; the edge holds customer state.

**SR-MPLS vs SRv6?**
SR-MPLS: segments are MPLS labels, data plane is MPLS. SRv6: segments are IPv6
addresses (locator + function), data plane is native IPv6 — no MPLS. One SRv6 SID can
carry both reachability and a service instruction.

**What does a uDT4 (End.DT4) SID do?**
It's the egress PE's L3VPN service SID: "decapsulate the outer IPv6 and do an IPv4
lookup in this VRF." The ingress encapsulates toward it; the egress delivers into the
right VRF.

**Why didn't the customer notice the MPLS→SRv6 change?**
The service (VRF, RD/RT, PE-CE peering, customer addressing) is independent of the
transport. Only the underlay encapsulation changed. CE1↔CE2 reachability is identical
either way — the L3VPN is transport-agnostic.

**What's the difference between the L3VPN and L2VPN service SIDs in SRv6?**
`uDT4` (L3VPN) means "decapsulate, then do an IPv4 route lookup in the VRF." `uDX2`
(EVPN-VPWS / L2VPN) means "decapsulate, then hand the Ethernet frame straight to this
L2 port." Same SRv6 encapsulation machinery; the only difference is the egress
instruction — route vs deliver-to-port.

**How is EVPN-VPWS signalled, and how do you know the L2 service really works?**
EVPN signals the pseudowire over BGP using the `l2vpn evpn` address-family
(Route-Type 1, Ethernet Auto-Discovery). You know it's truly Layer 2 because the two
CEs are in the same subnet and each learns the *other's MAC address* across the routed
core — something impossible if the core were routing between them.
