# Segment Routing (SR-MPLS) on Cisco IOS XRv9000 — EVE-NG Lab

A hands-on lab building a service-provider style core that forwards MPLS traffic
using **Segment Routing (SR-MPLS)** — no LDP, no RSVP-TE. The lab progresses from a
plain IGP up to traffic engineering and L3VPN services riding on an SR underlay,
mirroring how modern mobile/transport networks are built.

> **Goal:** learn SR the way operators actually deploy it, and produce a clean,
> reproducible writeup. Every phase lists *what* to configure, *why* it matters,
> and *how* to verify it.

---

## Skills demonstrated

- IS-IS as a service-provider IGP (Level-2, wide metrics)
- Segment Routing MPLS: SRGB, prefix-SIDs, adjacency-SIDs
- Fast convergence with **TI-LFA** (sub-50 ms reroute)
- **SR-TE** explicit-path policies and traffic steering
- L3VPN (MP-BGP VPNv4) over an SR transport
- Intro to **SRv6**
- Lab build & automation in EVE-NG with Cisco IOS XRv9000

---

## Lab environment

| Component | Detail |
|---|---|
| Emulator | EVE-NG (Community/Pro) |
| Node image | Cisco IOS XRv9000 (`xrv9k-fullk9-7.x`, reduced-resource build) |
| Core/PE nodes | 4 × XRv9000 (R1–R4), 4 vCPU, 8–16 GB RAM each |
| CE nodes | 2 × lightweight IOS (IOL/vIOS) |
| IGP | IS-IS Level-2-only |
| Transport | SR-MPLS (default SRGB 16000–23999) |

> **Resource note:** XRv9000 is RAM-hungry. If your host is tight, build R1/R2/R4
> first (phases 1–3) and add R3 later for the SR-TE path-diversity work.

---

## Topology

```
                 +-------+
        +--------|  R2   |--------+
        |        |  (P)  |        |
        |        +---+---+        |
        |            | cross-     |
   +----+---+        | link    +--+-----+
   |  R1    |        |         |  R4    |
   |  (PE)  |        |         |  (PE)  |
   +----+---+        |         +--+-----+
        |        +---+---+        |
        +--------|  R3   |--------+
        |        |  (P)  |        |
        |        +-------+        |
     +--+--+                   +--+--+
     | CE1 |                   | CE2 |
     +-----+                   +-----+
```

A "diamond": two equal-cost paths between the PEs (R1↔R2↔R4 and R1↔R3↔R4), plus an
R2–R3 cross-link that gives a loop-free backup so TI-LFA and SR-TE have something
interesting to do.

### Addressing & SID plan

| Node | Role | Loopback0 | Prefix-SID index | Expected SID label |
|---|---|---|---|---|
| R1 | PE | 1.1.1.1/32 | 1 | 16001 |
| R2 | P  | 2.2.2.2/32 | 2 | 16002 |
| R3 | P  | 3.3.3.3/32 | 3 | 16003 |
| R4 | PE | 4.4.4.4/32 | 4 | 16004 |

| Link | Subnet | A-end (intf, IP) | B-end (intf, IP) |
|---|---|---|---|
| R1–R2 | 10.12.0.0/30 | R1 Gi0/0/0/1 .1 | R2 Gi0/0/0/0 .2 |
| R1–R3 | 10.13.0.0/30 | R1 Gi0/0/0/3 .1 | R3 Gi0/0/0/2 .2 |
| R2–R3 (cross-link) | 10.23.0.0/30 | R2 Gi0/0/0/1 .1 | R3 Gi0/0/0/0 .2 |
| R2–R4 | 10.24.0.0/30 | R2 Gi0/0/0/3 .1 | R4 Gi0/0/0/2 .2 |
| R3–R4 | 10.34.0.0/30 | R3 Gi0/0/0/4 .1 | R4 Gi0/0/0/3 .2 |
| CE1–R1 (VRF CUST-A) | 192.168.11.0/30 | CE1 Gi0/0/0/1 .2 | R1 Gi0/0/0/0 .1 |
| CE2–R4 (VRF CUST-A) | 192.168.44.0/30 | CE2 Gi0/0/0/0 .2 | R4 Gi0/0/0/1 .1 |

**Customer (L3VPN) addressing:** VRF `CUST-A`, RD/RT `100:1`. CE1 AS 65001, Lo0 `11.11.11.11`. CE2 AS 65002, Lo0 `22.22.22.22`. Provider AS 100.

Full device configs are in [`configs/`](configs/).

---

## Phase 1 — IS-IS baseline

**Objective:** every loopback reachable via IS-IS L2 before any SR is enabled.
This is the foundation — if the IGP isn't clean, nothing above it will be.

Key points: `is-type level-2-only`, `metric-style wide` (mandatory for SR later),
point-to-point on the core links so adjacencies form fast.

**Verify**

```
show isis neighbors
show route isis
ping 4.4.4.4 source 1.1.1.1
```

You should see full IS-IS adjacencies and reach every loopback. No MPLS yet.

---

## Phase 2 — Enable SR-MPLS

**Objective:** turn the IGP into a label-switched core with **zero** label-distribution
protocol running.

**Why it matters:** in classic MPLS you need LDP or RSVP-TE to hand out labels. With
SR, the IGP itself advertises a globally-agreed label (the *prefix-SID*) for each
loopback. Fewer protocols, simpler core, easier troubleshooting — the main reason
operators moved to SR.

Two lines do the work: `segment-routing mpls` under the IPv4 address-family, and
`prefix-sid index N` under each loopback.

```
router isis CORE
 is-type level-2-only
 net 49.0001.0000.0000.0001.00
 address-family ipv4 unicast
  metric-style wide
  segment-routing mpls
 !
 interface Loopback0
  passive
  address-family ipv4 unicast
   prefix-sid index 1
  !
 !
 interface GigabitEthernet0/0/0/0
  point-to-point
  address-family ipv4 unicast
  !
 !
```

**Verify**

```
show isis segment-routing label table
show mpls forwarding
show cef 4.4.4.4/32 detail
traceroute 4.4.4.4 source 1.1.1.1
```

Expected: prefix-SIDs in the 16001–16004 range, an MPLS forwarding entry per remote
loopback, and a traceroute that shows MPLS labels in transit — all with no LDP
anywhere in the config.

> **Understand, don't just paste:** `show cef 4.4.4.4/32 detail` shows the *label
> stack* R1 imposes to reach R4. Be able to explain why it's a single label here, and
> when it would become a stack.

---

## Phase 3 — Adjacency-SIDs + TI-LFA fast reroute

**Objective:** protect every prefix with a pre-computed backup path so a link failure
reroutes in well under 50 ms.

**Why it matters:** TI-LFA is one of SR's headline features and a very common
interview topic. Because SR can express *any* backup path as a label stack, TI-LFA
guarantees a loop-free alternate even in topologies where classic LFA fails.

Add under each core interface's address-family:

```
 interface GigabitEthernet0/0/0/0
  point-to-point
  address-family ipv4 unicast
   fast-reroute per-prefix
   fast-reroute per-prefix ti-lfa
  !
 !
```

**Verify**

```
show isis fast-reroute summary
show cef 4.4.4.4/32 detail        ! note the backup (repair) path + label
```

Then prove it: start a ping flood from CE1 to CE2, `shutdown` one core link, and
confirm only a packet or two is lost.

> **Check your understanding:** look at the *repair label stack* in the CEF output.
> Which node does the backup path steer through, and why that one?

---

## Phase 4 — SR-TE explicit-path policy

**Objective:** force traffic from R1 to R4 down the *non-shortest* path using an SR-TE
policy and a segment list.

**Why it matters:** this is traffic engineering without RSVP-TE state in the core. The
headend pushes a label stack describing the path; transit routers just forward on
labels. This is how operators do latency-based / disjoint-path steering today.

Target config (headend R1, "scenic" path R1→R2→R3→R4 via node-SIDs):

```
segment-routing
 traffic-eng
  segment-list SCENIC-R1-TO-R4
   index 10 mpls label 16002     ! prefix-SID of R2
   index 20 mpls label 16003     ! prefix-SID of R3
   index 30 mpls label 16004     ! prefix-SID of R4
  !
  policy R1-TO-R4-SCENIC
   color 100 end-point ipv4 4.4.4.4
   autoroute
    include ipv4 4.4.4.4/32       ! lab steering; production uses color/ODN
   !
   candidate-paths
    preference 100
     explicit segment-list SCENIC-R1-TO-R4
    !
   !
  !
 !
!
```

Defining the policy isn't enough — traffic only follows it once steered. `autoroute`
is the lab-simple method; in production, color-based BGP automated steering (ODN)
pulls coloured routes into the matching policy.

**Verify**

```
show segment-routing traffic-eng policy
traceroute 4.4.4.4 source 1.1.1.1     ! should now traverse R3, not R2
```

---

## Phase 5 — L3VPN (MP-BGP VPNv4) over SR

**Objective:** carry a real customer VRF (CE1 ↔ CE2) across the SR core.

**Why it matters:** SR is just the *underlay*. The point is to ride services on top.
This is the bread-and-butter operator use case and proves the whole stack works
end to end.

Building blocks:
- VRF (e.g. `CUST-A`) on R1 and R4, with the CE-facing interface in the VRF
- MP-BGP VPNv4 session between R1 and R4 loopbacks
- PE–CE routing (static or eBGP) to exchange customer prefixes

**Verify**

```
show bgp vpnv4 unicast summary
show route vrf CUST-A
! from CE1:
ping <CE2 address>
```

End-to-end CE-to-CE ping across the SR core = lab success.

---

## Phase 6 — SRv6 (stretch goal)

**Objective:** repeat a simple L3VPN, but over **SRv6** instead of SR-MPLS.

**Why it matters:** mobile transport is moving toward SRv6 (and uSID). Even basic
familiarity — locators, the IPv6-native data plane, no MPLS label stack — stands out
on a CV for an operator role.

Building blocks: enable `segment-routing srv6`, define a locator per node, advertise
it in IS-IS, then build the VPN over SRv6.

**Verify**

```
show segment-routing srv6 locator
show route
```

---

## Results

- [x] **Phase 1** — IS-IS adjacencies up, loopbacks reachable
- [x] **Phase 2** — SR-MPLS labels (16001–16004), traceroute shows labels
- [x] **Phase 3** — TI-LFA backup path verified, zero loss on link failure
- [x] **Phase 4** — SR-TE policy steers traffic R1→R2→R3→R4 (against IGP)
- [x] **Phase 5** — CE1↔CE2 L3VPN ping success across the SR core
- [ ] **Phase 6** — L3VPN over SRv6 _(stretch goal — not yet built)_

### Phase 1 — IS-IS baseline

Equal-cost paths to R4 via both R2 and R3 confirm the diamond is working:

```
R1# show route isis
i L2 4.4.4.4/32 [115/20] via 10.13.0.2, GigabitEthernet0/0/0/3
                [115/20] via 10.12.0.2, GigabitEthernet0/0/0/1
R1# ping 4.4.4.4 source 1.1.1.1
Success rate is 100 percent (5/5)
```

### Phase 2 — SR-MPLS enabled

Globally-agreed prefix-SID labels, and a traceroute now carrying an MPLS label —
with no LDP or RSVP anywhere in the config:

```
R1# show isis segment-routing label table
16001  1.1.1.1/32   SPF
16002  2.2.2.2/32   SPF
16003  3.3.3.3/32   SPF
16004  4.4.4.4/32   SPF

R1# traceroute 4.4.4.4 source 1.1.1.1
 1  10.12.0.2 [MPLS: Label 16004 Exp 0]
 2  10.24.0.2
```

### Phase 3 — TI-LFA fast reroute

100% protection coverage; a pre-computed backup installed for single-path prefixes.
(The backup shows as `Local-LFA` because the R2–R3 cross-link provides a directly
loop-free alternate — TI-LFA only builds an explicit repair tunnel when none exists.)

```
R1# show isis fast-reroute summary
  All paths protected ...  Total 6   Protection coverage 100.00%

R1# show cef 2.2.2.2/32 detail
   via 10.13.0.2 ... backup (Local-LFA)   labels imposed {16002}
   via 10.12.0.2 ... protected            labels imposed {ImplNull}

R1# ping 2.2.2.2 source 1.1.1.1 count 1000   (link shut mid-test)
Success rate is 100 percent (1000/1000)
```

### Phase 4 — SR-TE traffic engineering

Traffic forced onto the 3-hop "scenic" path the IGP would never choose. Watch the
label stack shrink as each node-SID is consumed:

```
R1# show route 4.4.4.4
  4.4.4.4, via srte_c_100_ep_4.4.4.4

R1# traceroute 4.4.4.4 source 1.1.1.1
 1  10.12.0.2 [MPLS: Labels 16003/16004]   <- R2
 2  10.23.0.2 [MPLS: Label 16004]          <- R3 (cross-link)
 3  10.34.0.2                              <- R4
```

### Phase 5 — L3VPN over SR

Customer CE1 reaches CE2's loopback across the SR core, inside VRF `CUST-A`. The VPN
route resolves its next-hop *in the global table* (`nexthop in vrf default`) — the
two-label (transport + VPN) separation that keeps the core stateless:

```
CE1# ping 22.22.22.22 source 11.11.11.11
Success rate is 100 percent (5/5)

R1# show bgp vpnv4 unicast        (RD 100:1, vrf CUST-A)
*> 11.11.11.11/32   192.168.11.2        0  65001 i
*>i22.22.22.22/32   4.4.4.4        100   0  65002 i

R1# show route vrf CUST-A
B  22.22.22.22/32 [200/0] via 4.4.4.4 (nexthop in vrf default)
```

---

## Concepts to be able to explain (interview prep)

- The difference between a **prefix-SID** (global) and an **adjacency-SID** (local)
- What the **SRGB** is and why a consistent block matters across the domain
- Why SR removes the need for LDP/RSVP-TE in the core
- How **TI-LFA** guarantees a loop-free backup where classic LFA cannot
- How an **SR-TE policy** programs a path as a label stack at the headend
- Where **SR-MPLS** ends and **SRv6** begins, and why operators are interested in SRv6

---

## References

- Cisco IOS XR Segment Routing Configuration Guide
- EVE-NG documentation — adding Cisco IOS XRv9000
- IETF RFC 8402 (Segment Routing Architecture)

---

*Lab built and documented by BO SAM ATH. Feedback welcome via issues/PRs.*
