# Phase 7 — EVPN-VPWS (L2VPN) over SRv6 (uDX2): Verification Log

**Date:** 2026-06-24
**Objective:** Carry a point-to-point Layer 2 service (E-Line) between CE1 and CE2 so
they sit in the same subnet (172.16.0.0/24) and behave as if directly cabled, while
the routed SR core sits invisibly between them — signalled by EVPN and encapsulated to
the egress PE's **uDX2** SID (the L2 sibling of Phase 6's uDT4).

**Result:** ✅ **PASS** — xconnect `CE1-CE2-L2` up with `Encapsulation SRv6`, uDX2
local/remote SIDs resolved, EVI 200 established, **CE1→CE2 ping 5/5**, and CE1 learned
**CE2's MAC dynamically** — a genuine Layer 2 service over the SRv6 core.

---

## Commands run

```
show l2vpn xconnect detail            ! PW state + uDX2 SIDs (R1)
show evpn evi detail                  ! EVI 200 (R1)
show segment-routing srv6 sid         ! uDX2 alongside uN/uA/uDT4 (R1)
ping 172.16.0.2                       ! from CE1 — same-subnet L2 reachability
show arp                              ! from CE1 — proves MAC learning
```

---

## Pseudowire state + uDX2 SIDs (R1)

```
RP/0/RP0/CPU0:R1#show l2vpn xconnect detail
Group EVPN-VPWS, XC CE1-CE2-L2, state is up; Interworking none
  AC: GigabitEthernet0/0/0/2, state is up
    Type Ethernet ; MTU 1500
  EVPN: neighbor ::ffff:10.0.0.1, PW ID: evi 200, ac-id 1, state is up ( established )
    Encapsulation SRv6
      SRv6              Local                        Remote
      ----------------  ---------------------------- --------------------------
      uDX2              fcbb:bb00:1:e000::           fcbb:bb00:4:e000::
      AC ID             1                            1
      Locator           MAIN                         N/A
      Locator Resolved  Yes                          N/A
      SRv6 Headend      H.Encaps.L2.Red              N/A
```

## EVI 200 (R1)

```
RP/0/RP0/CPU0:R1#show evpn evi detail
VPN-ID     Encap      Bridge Domain                Type
---------- ---------- ---------------------------- -------------------
200        SRv6       VPWS:200                     VPWS (vlan-unaware)
   RD Auto  : (auto) 1.1.1.1:200
   RT Auto  : 100:200
   Route Targets in Use           Type
   100:200                        Import
   100:200                        Export
```

## SID table — one locator carries everything (R1)

```
RP/0/RP0/CPU0:R1#show segment-routing srv6 sid
SID                         Behavior          Context
--------------------------  ----------------  --------------------
fcbb:bb00:1::               uN (PSP/USD)      'default':1          <- transport node SID
fcbb:bb00:1:e000::          uDX2              200:1                <- L2VPN (this phase)
fcbb:bb00:1:e001::          uA (PSP/USD)      [Gi0/0/0/1 ...]      <- adjacency SID
fcbb:bb00:1:e002::          uA (PSP/USD)      [Gi0/0/0/3 ...]      <- adjacency SID
fcbb:bb00:1:e003::          uDT4              'CUST-A'             <- L3VPN (Phase 6)
```

## Layer-2 proof — CE1 to CE2 (CE1)

```
RP/0/RP0/CPU0:CE1#ping 172.16.0.2
!!!!!  Success rate is 100 percent (5/5), round-trip min/avg/max = 11/24/66 ms

RP/0/RP0/CPU0:CE1#show arp
Address         Age        Hardware Addr   State      Type  Interface
172.16.0.1      -          5000.0001.0005  Interface  ARPA  Gi0/0/0/2
172.16.0.2      00:00:03   5000.0006.0004  Dynamic    ARPA  Gi0/0/0/2   <- CE2's MAC, learned
```

---

## Analysis

- **The pseudowire is up over SRv6.** The xconnect `CE1-CE2-L2` shows `state is up`,
  the EVPN PW `( established )`, and `Encapsulation SRv6` with
  `SRv6 Headend H.Encaps.L2.Red` — R1 encapsulates the *whole Ethernet frame* into
  IPv6 toward R4's uDX2 SID. ✔
- **uDX2 = the L2 sibling of uDT4.** Local `fcbb:bb00:1:e000::` ↔ remote
  `fcbb:bb00:4:e000::`. `uDX2` means "decapsulate and **cross-connect to the L2 port**"
  — versus `uDT4` (Phase 6) which means "decapsulate and **do an IPv4 VRF lookup**".
  Same locator, same encap machinery; only the egress instruction differs. ✔
- **EVPN signals it.** EVI **200**, type **VPWS (vlan-unaware)**, auto RD `1.1.1.1:200`,
  auto RT `100:200` import/export — BGP `l2vpn evpn` (Route-Type 1 / Ethernet A-D)
  brought the PW up, no manual PW config. ✔
- **The capstone — one locator carries the whole stack.** `MAIN` simultaneously holds
  `uN` (transport), `uA` (adjacency), `uDT4` (L3VPN), and `uDX2` (L2VPN). L2 and L3
  services coexist on one four-node core over a single SRv6 locator. ✔
- **It is genuinely Layer 2.** CE1 reaches CE2 in the same `172.16.0.0/24` subnet with
  no IP config on the PEs for this service, and CE1's ARP table shows **CE2's MAC
  `5000.0006.0004` learned dynamically** across the pseudowire. MAC learning end to end
  = a true E-Line, not routing. ✔

**Conclusion:** EVPN-VPWS L2VPN over SRv6 verified end to end. **All 7 phases complete
and validated** — IS-IS → SR-MPLS → TI-LFA → SR-TE → L3VPN(SR-MPLS) → L3VPN(SRv6) →
EVPN-VPWS(SRv6), L2 and L3 services on one SR core.
