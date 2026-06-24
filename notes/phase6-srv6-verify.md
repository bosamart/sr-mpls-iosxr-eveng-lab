# Phase 6 — L3VPN over SRv6 (uDT4): Verification Log

**Date:** 2026-06-24
**Objective:** Carry the *same* `CUST-A` L3VPN over **SRv6** instead of SR-MPLS. A
single IPv6 SID should replace both the transport label and the VPN label — locator
(reach the egress PE) and function (VRF lookup) in one address.

**Result:** ✅ **PASS** — SRv6 locator `MAIN` up; the SID table shows the node SID
(`uN`), adjacency SIDs (`uA`), the VPN service SID (`uDT4` for CUST-A) — and the CEF
entry for the remote customer prefix shows a real **SRv6 H.Encaps** toward R4's uDT4
SID, no MPLS.

---

## Commands run

```
show segment-routing srv6 locator
show segment-routing srv6 sid
show cef vrf CUST-A 22.22.22.22 detail
```

---

## Locator + SID table (R1)

```
RP/0/RP0/CPU0:R1#show segment-routing srv6 locator
Name                  ID       Algo  Prefix                    Status   Flags
--------------------  -------  ----  ------------------------  -------  --------
MAIN                  1        0     fcbb:bb00:1::/48          Up       U

RP/0/RP0/CPU0:R1#show segment-routing srv6 sid
*** Locator: 'MAIN' ***
SID                         Behavior          Context                     Owner        State
--------------------------  ----------------  --------------------------  -----------  -----
fcbb:bb00:1::               uN (PSP/USD)      'default':1                 sidmgr       InUse
fcbb:bb00:1:e000::          uDX2              200:1                       l2vpn_srv6   InUse   <- Phase 7 L2VPN
fcbb:bb00:1:e001::          uA (PSP/USD)      [Gi0/0/0/1, Link-Local]:0   isis-CORE    InUse
fcbb:bb00:1:e002::          uA (PSP/USD)      [Gi0/0/0/3, Link-Local]:0   isis-CORE    InUse
fcbb:bb00:1:e003::          uDT4              'CUST-A'                    bgp-100      InUse   <- L3VPN service
```

## SRv6 encapsulation for the remote customer prefix (R1)

```
RP/0/RP0/CPU0:R1#show cef vrf CUST-A 22.22.22.22 detail
22.22.22.22/32, version 6, SRv6 Headend ...
  [0] via fcbb:bb00:4::/128, recursive
    next hop fcbb:bb00:4::/128 via fcbb:bb00:4::/48
    SRv6 H.Encaps.Red SID-list {fcbb:bb00:4:e003::}        <- encap toward R4's uDT4
    Hash  OK  Interface                 Address
    0     Y   GigabitEthernet0/0/0/3    remote
    1     Y   GigabitEthernet0/0/0/1    remote
```

---

## Analysis

- **The data-plane proof — `SRv6 Headend` + `H.Encaps.Red`.** To reach CE2's
  `22.22.22.22/32`, R1 encapsulates the customer packet into an outer IPv6 header
  with destination **`fcbb:bb00:4:e003::`** — R4's **uDT4** service SID. No MPLS label
  stack: the locator `fcbb:bb00:4::/48` routes it to R4, and the uDT4 function tells R4
  to decapsulate and do the CUST-A IPv4 lookup. ✔
- **One SID carries reachability + service.** `uDT4` is the SRv6 form of `End.DT4`
  ("decapsulate + IPv4 VRF lookup"), allocated by BGP and bound to `'CUST-A'`. It
  replaces both the transport label and the VPN label from Phase 5. ✔
- **`uN` replaces the prefix-SID; `uA` replace the adjacency labels.** `fcbb:bb00:1::`
  (uN) is R1's node SID — the SRv6 equivalent of `16001`; the two `uA` are the
  per-link adjacency SIDs. ✔
- **Recursion + ECMP intact.** The encap next-hop `fcbb:bb00:4::/128` resolves
  recursively over both core interfaces (Gi0/0/0/1 and Gi0/0/0/3) — SRv6 still rides
  the IGP's equal-cost paths. ✔
- **Same VRF, two data planes.** `CUST-A` is now validated over **both** SR-MPLS
  (Phase 5) and SRv6 (Phase 6) — a transport-agnostic L3VPN. ✔
- **Foreshadowing Phase 7:** the SID table already carries a **`uDX2` (200:1)** entry
  (owner `l2vpn_srv6`) on the same locator — the EVPN-VPWS L2 service SID, verified
  next.

**Conclusion:** L3VPN over SRv6 verified, with real H.Encaps capture. Proceed to
Phase 7 (EVPN-VPWS L2VPN) — the uDX2 alongside this uDT4 on one locator.
