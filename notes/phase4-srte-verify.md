# Phase 4 — SR-TE Explicit-Path Policy: Verification Log

**Date:** 2026-06-24
**Objective:** Force traffic from R1 to R4 down the **non-shortest** "scenic" path
R1→R2→R3→R4 using an SR-TE policy with an explicit segment list (16002, 16003,
16004) — with zero config on the transit routers.

**Result:** ✅ **PASS** — the policy is **up/up**, the explicit segment-list is valid,
and a traceroute confirms the 3-hop scenic path with the label stack shrinking by one
SID at each hop.

---

## Commands run

```
show segment-routing traffic-eng policy
traceroute 4.4.4.4 source 1.1.1.1
```

---

## Policy state (R1)

```
RP/0/RP0/CPU0:R1#show segment-routing traffic-eng policy
Color: 100, End-point: 4.4.4.4
  Name: srte_c_100_ep_4.4.4.4
  Status:
    Admin: up  Operational: up for 00:07:26 (since Jun 24 17:47:10.455)
  Candidate-paths:
    Preference: 100 (configuration) (active)
      Name: R1-TO-R4-SCENIC
      Requested BSID: dynamic
      Constraints:
        Protection Type: protected-preferred
        Maximum SID Depth: 10
      Explicit: segment-list SCENIC-R1-TO-R4 (valid)
        Weight: 1, Metric Type: TE
          SID[0]: 16002
          SID[1]: 16003
          SID[2]: 16004
  Attributes:
    Binding SID: 24006
    Path Type: SRMPLSv4
```

## Data-path proof (R1)

```
RP/0/RP0/CPU0:R1#traceroute 4.4.4.4 source 1.1.1.1
 1  10.12.0.2 [MPLS: Labels 16003/16004 Exp 0] 9 msec  3 msec  4 msec
 2  10.23.0.2 [MPLS: Label 16004 Exp 0] 7 msec  4 msec  4 msec
 3  10.34.0.2 24 msec  *  6 msec
```

---

## Analysis

- **The headend imposes the whole path as a label stack.** At hop 1 R1 has pushed
  **{16003, 16004}** (it already consumed 16002 to reach R2). At hop 2 — R3, reached
  over the **10.23.0.2 cross-link** — only **{16004}** remains. At hop 3 the packet
  arrives at R4 as plain IP. The stack shrinks exactly one SID per hop. ✔
- **Transit routers hold zero TE state.** R2 and R3 just pop/forward on the SIDs they
  already advertise — no policy, no per-tunnel config. All intelligence sits at R1.
  This is the core advantage over RSVP-TE (where every hop holds PATH/RESV state). ✔
- **It's deliberately the longer path.** IGP shortest path to R4 is 2 hops; the policy
  forces the 3-hop R1→R2→R3→R4 route through the cross-link — the whole point of TE. ✔
- **Policy validity + constraints.** Segment-list shows **(valid)**, `Metric Type: TE`,
  `Maximum SID Depth: 10`, `Protection Type: protected-preferred`. **Binding SID 24006**
  is allocated — the label other nodes/services use to steer into this path. ✔

**Conclusion:** Explicit-path SR-TE steering verified end to end. Proceed to Phase 5
(L3VPN) to carry a real customer VRF over this SR transport.
