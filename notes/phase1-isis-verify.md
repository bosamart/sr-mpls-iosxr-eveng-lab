# Phase 1 — IS-IS Underlay: Verification Log

**Date:** 2026-06-24
**Objective:** Bring up IS-IS Level-2 on the 4-node diamond (R1–R4) so every
loopback is reachable, with equal-cost paths between the PEs — the clean IGP
foundation that SR-MPLS rides on.

**Result:** ✅ **PASS** — full L2 adjacencies on every core link, all loopbacks in the
routing table, R1↔R4 over two equal-cost paths, and bidirectional loopback ping 5/5.

---

## Commands run

```
show isis neighbors      ! per core router
show route isis          ! loopbacks + link prefixes
ping <remote-lo> source <local-lo>
```

---

## Adjacencies

```
RP/0/RP0/CPU0:R1#show isis neighbors
System Id      Interface        SNPA           State Holdtime Type IETF-NSF
R2             Gi0/0/0/1        *PtoP*         Up    22       L2   Capable
R3             Gi0/0/0/3        *PtoP*         Up    28       L2   Capable
Total neighbor count: 2

RP/0/RP0/CPU0:R4#show isis neighbors
System Id      Interface        SNPA           State Holdtime Type IETF-NSF
R2             Gi0/0/0/2        *PtoP*         Up    22       L2   Capable
R3             Gi0/0/0/3        *PtoP*         Up    22       L2   Capable
Total neighbor count: 2
```

## Routing table — ECMP proof

```
RP/0/RP0/CPU0:R1#show route isis
i L2 4.4.4.4/32 [115/20] via 10.13.0.2, GigabitEthernet0/0/0/3
                [115/20] via 10.12.0.2, GigabitEthernet0/0/0/1
i L2 2.2.2.2/32 [115/20] via 10.13.0.2, GigabitEthernet0/0/0/3 (!)
                [115/10] via 10.12.0.2, GigabitEthernet0/0/0/1
i L2 3.3.3.3/32 [115/10] via 10.13.0.2, GigabitEthernet0/0/0/3
                [115/20] via 10.12.0.2, GigabitEthernet0/0/0/1 (!)

RP/0/RP0/CPU0:R4#show route isis
i L2 1.1.1.1/32 [115/20] via 10.34.0.1, GigabitEthernet0/0/0/3
                [115/20] via 10.24.0.1, GigabitEthernet0/0/0/2
```

## Reachability — bidirectional

```
RP/0/RP0/CPU0:R1#ping 4.4.4.4 source 1.1.1.1
!!!!!  Success rate is 100 percent (5/5), round-trip min/avg/max = 3/4/5 ms

RP/0/RP0/CPU0:R4#ping 1.1.1.1 source 4.4.4.4
!!!!!  Success rate is 100 percent (5/5), round-trip min/avg/max = 6/21/73 ms
```

---

## Analysis

- **All adjacencies point-to-point / L2.** `is-type level-2-only` + `point-to-point`
  on the core interfaces — fast adjacencies, no DIS/pseudonode overhead. ✔
- **PE-to-PE ECMP confirmed.** R1 reaches `4.4.4.4/32` over **two** equal-cost paths
  (metric 20, via R3 at 10.13.0.2 and via R2 at 10.12.0.2); R4 mirrors this back to
  `1.1.1.1/32`. That's the diamond providing two disjoint PE paths — what TI-LFA and
  SR-TE will exploit later. ✔
- **Single-hop metrics correct.** `2.2.2.2` (R2) is metric **10** via the direct link
  but **20** via the far side; `(!)` marks the non-best/backup path. ✔
- **Clean IGP state.** `4.4.4.4` resolves via plain IS-IS ECMP here (no SR-TE policy
  yet) — a true pre-SR baseline. ✔
- **`metric-style wide` in place** (mandatory before SR-MPLS in Phase 2).

**Conclusion:** IGP underlay is clean and provides ECMP between the PEs. Proceed to
Phase 2 (enable SR-MPLS) — the labels ride on exactly these paths.
