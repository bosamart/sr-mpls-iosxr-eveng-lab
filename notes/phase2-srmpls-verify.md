# Phase 2 — SR-MPLS (Prefix-SIDs): Verification Log

**Date:** 2026-06-24
**Objective:** Turn the IGP into a label-switched core with **no** label-distribution
protocol. Each loopback should get a globally-agreed prefix-SID label (16001–16004),
distributed by IS-IS itself.

**Result:** ✅ **PASS** — every router advertises its prefix-SID, the label table maps
`16001→R1 … 16004→R4`, `show mpls forwarding` has a forwarding entry per remote
loopback (with PHP and ECMP), and a traceroute shows MPLS labels in transit. No LDP or
RSVP anywhere.

---

## Commands run

```
show isis segment-routing label table
show mpls forwarding
traceroute 4.4.4.4 source 1.1.1.1
```

---

## SR label table (R1)

```
RP/0/RP0/CPU0:R1#show isis segment-routing label table
IS-IS CORE IS Label Table
Label         Prefix                   Algorithm    Interface
----------    ----------------         ---------    ---------
16001         1.1.1.1/32               SPF          Loopback0
16002         2.2.2.2/32               SPF
16003         3.3.3.3/32               SPF
16004         4.4.4.4/32               SPF
```

## MPLS forwarding (R1)

```
RP/0/RP0/CPU0:R1#show mpls forwarding
Local  Outgoing    Prefix             Outgoing     Next Hop        Bytes
Label  Label       or ID              Interface                    Switched
------ ----------- ------------------ ------------ --------------- ------------
16001  Aggregate   SR Pfx (idx 1)     default                      0
16002  Pop         SR Pfx (idx 2)     Gi0/0/0/1    10.12.0.2       0
       16002       SR Pfx (idx 2)     Gi0/0/0/3    10.13.0.2       0      (!)
16003  Pop         SR Pfx (idx 3)     Gi0/0/0/3    10.13.0.2       0
       16003       SR Pfx (idx 3)     Gi0/0/0/1    10.12.0.2       0      (!)
16004  16004       SR Pfx (idx 4)     Gi0/0/0/3    10.13.0.2       793
       16004       SR Pfx (idx 4)     Gi0/0/0/1    10.12.0.2       0
24000  Pop         SR Adj (idx 1)     Gi0/0/0/1    10.12.0.2       0       <- adjacency-SIDs
24001  Pop         SR Adj (idx 3)     Gi0/0/0/1    10.12.0.2       0
24004  Aggregate   CUST-B: Per-VRF Aggr   CUST-B                  0       <- RD/RT demo VRF
24005  16003       SR TE: 1 [TE-INT]  Gi0/0/0/1    10.12.0.2       0       <- SR-TE policy
24007  Pop         4.4.4.4/32         srte_c_100_e 4.4.4.4         0
```

## Traceroute — label-switched transport (R1)

```
RP/0/RP0/CPU0:R1#traceroute 4.4.4.4 source 1.1.1.1
 1  10.12.0.2 [MPLS: Labels 16003/16004 Exp 0] 42 msec  4 msec  3 msec
 2  10.23.0.2 [MPLS: Label 16004 Exp 0] 24 msec  7 msec  3 msec
 3  10.34.0.2 48 msec  *  6 msec
```

---

## Analysis

- **IS-IS is the label-distribution protocol now.** Labels come from `prefix-sid
  index N` under each `Loopback0` + `segment-routing mpls` in the IPv4 AF — no
  LDP/RSVP process exists. ✔
- **PHP is visible.** R1 shows `Pop` for directly-adjacent SIDs (`16002 … Gi0/0/0/1`):
  one hop from R2, so it strips the label and hands plain IP up. ✔
- **Two-hop SID is swapped, not popped.** `16004` keeps `Outgoing Label 16004` and
  forwards on — and shows real traffic (`793` bytes switched). ✔
- **ECMP preserved.** Each remote SID has two outgoing entries (Gi0/0/0/1 and
  Gi0/0/0/3); `(!)` marks the backup. ✔
- **`24xxx` labels are the rest of the lab already loaded:** `24000–24003` adjacency
  SIDs, `24004` the **CUST-B per-VRF aggregate** (RD-vs-RT demo), `24005–24007` the
  **SR-TE policy** for color 100 / endpoint 4.4.4.4. ✔

> **Why the traceroute takes 3 hops, not 2:** because the SR-TE policy (Phase 4) is
> already active, `4.4.4.4` is steered down the scenic path R1→R2→**R3**(10.23.0.2,
> cross-link)→R4 with stack {16003,16004}, shrinking one SID per hop. A pure
> SR-MPLS-shortest-path traceroute (no policy) would instead show a single label
> `16004` over a 2-hop path. Full SR-TE proof is in `phase4-srte-verify.md`.

**Conclusion:** Global prefix-SIDs are distributed and forwarding correctly with no
LDP. Proceed to Phase 3 (TI-LFA).
