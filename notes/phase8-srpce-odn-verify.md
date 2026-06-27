# Phase 8 — SR-PCE + ODN: Verification Log

**Date:** 2026-06-27
**Objective:** Prove controller-driven SR-TE end to end — a **colored** L3VPN route makes
the headend ask the **SR-PCE** for a path, an SR-TE policy is **auto-created** (no
hand-built segment-list), and real customer traffic rides it.

**Result:** ✅ **PASS** — PCE learned the full topology via BGP-LS, R1's PCEP session is
up, the colored route (`44.44.44.44`, `Color:100`) triggered an **ODN** policy that the
**PCE computed** (`Dynamic (pce 5.5.5.5) valid`), and **CE3 → CE4 ping is 5/5** over it.

> **Setup for this run:** SR-PCE node `5.5.5.5` (leaf off R2, BGP-LS producer). Light
> CSR 1000v customer sites **CE3** (`33.33.33.33`, AS 65003 @ R1) and **CE4**
> (`44.44.44.44`, AS 65004 @ R4) in VRF CUST-A. R4 colors CUST-A routes **color 100**.
> Powered: R1–R4 + PCE + CE3 + CE4 (~48 GB — see `docs/EVE-NG-RESOURCES.md`).

---

## Commands run

```
! PCE
show pce ipv4 topology summary
show pce lsp detail
! R1 (PCC / headend)
show segment-routing traffic-eng pcc ipv4 peer
show bgp vrf CUST-A 44.44.44.44/32 detail
show segment-routing traffic-eng policy
show bgp vrf CUST-A neighbor 192.168.30.2 advertised-routes
! CE3 (customer)
show ip bgp summary
ping 44.44.44.44 source 33.33.33.33
```

---

## 1. PCE has the topology (BGP-LS feed works)

```
PCE# show pce ipv4 topology summary
  Topology nodes:        5
  Prefixes:              5
  Prefix SIDs:  Total:   5
  Links:        Total:  12
  Topology Ready Summary:  Ready: yes   PCEP allowed: yes
```

## 2. PCEP session up (R1 ↔ PCE)

```
R1# show segment-routing traffic-eng pcc ipv4 peer
  Peer address: 5.5.5.5,  Precedence: 10, (best PCE)
  State up
  Capabilities: Stateful, Update, Segment-Routing, Instantiation, SRv6
```

## 3. The colored service route reaches the headend

```
R1# show bgp vrf CUST-A 44.44.44.44/32 detail
  65004
    4.4.4.4 C:100 (bsid:24006) (metric 20) from 4.4.4.4 (4.4.4.4)
      Extended community: Color:100 RT:100:1
      SR policy color 100, up, registered, bsid 24006
```

## 4. ODN auto-created the policy — computed by the PCE

```
R1# show segment-routing traffic-eng policy
Color: 100, End-point: 4.4.4.4   Name: srte_c_100_ep_4.4.4.4
  Status: Admin: up  Operational: up
    Preference: 100 (BGP ODN) (active)
      Symbolic name: bgp_c_100_ep_4.4.4.4_discr_100
      Dynamic (pce 5.5.5.5) (valid)
        Metric Type: IGP,  Path Accumulated Metric: 20
          SID[0]: 16004 [Prefix-SID, 4.4.4.4]
  Binding SID: 24006
```

```
PCE# show pce lsp detail
PCC 1.1.1.1:  Tunnel Name: bgp_c_100_ep_4.4.4.4_discr_100   Color: 100
   State: Admin up, Operation active   Setup type: Segment Routing
   Computed path: (Local PCE)   Metric type: IGP, Accumulated Metric 20
      SID[0]: Node, Label 16004, Address 4.4.4.4
   SR Policy Association Group:  Color: 100, Endpoint: 4.4.4.4
```

## 5. The customer learns the route and reaches the far site

```
R1# show bgp vrf CUST-A neighbor 192.168.30.2 advertised-routes
  44.44.44.44/32   192.168.30.1   4.4.4.4   100 65004i
  ... (5 prefixes advertised to CE3)

CE3# show ip bgp summary
  Neighbor      V    AS  ...  Up/Down   State/PfxRcd
  192.168.30.1  4   100  ...  00:00:12  5

CE3# ping 44.44.44.44 source 33.33.33.33
  !!!!!  Success rate is 100 percent (5/5), round-trip min/avg/max = 6/11/27 ms
```

---

## Analysis

- **The PCE has a live map.** `Topology Ready: yes`, 5 nodes / 12 links — R2's
  `distribute link-state` → BGP-LS fed the controller the whole IS-IS topology. ✔
- **The headend is a PCC.** PCEP to `5.5.5.5` is `up` with Stateful + Instantiation +
  Segment-Routing capabilities — R1 can ask the PCE to compute and can host
  PCE-initiated LSPs. ✔
- **Color is the trigger.** CE4's `44.44.44.44` arrives at R1 carrying `Color:100`
  (set by R4). R1 maps it straight onto SR policy color 100 (`SR policy color 100, up,
  registered`). No per-destination config — the *color* did it. ✔
- **The path was computed by the PCE, not by hand.** The active candidate is
  `(BGP ODN) … Dynamic (pce 5.5.5.5) (valid)` — R1 delegated the computation over PCEP
  and the PCE returned `SID 16004` (reach R4). The old Phase 4 explicit segment-list is
  gone; this is fully automatic. ✔
- **End to end.** CE3 (a real customer) pings CE4's loopback **5/5** across the SR core,
  riding the ODN/PCE policy. ✔

> **Path note:** the ODN template optimizes on **IGP metric**, so the PCE chose the
> *shortest* path (`SID 16004`, metric 20). Correct behavior — but a traceroute looks
> like normal routing because shortest *is* normal. Forcing a non-shortest path via an
> affinity constraint was attempted and found **not exposed under `on-demand color` on this
> XRv9000 build** (see "Note on path constraints" in
> [`../docs/PHASE8-SR-PCE-ODN.md`](../docs/PHASE8-SR-PCE-ODN.md)).

**Conclusion:** Controller-driven SR-TE verified end to end — colored route →
PCE-computed policy → customer traffic on it, fully automatic. Phase 8 complete.
