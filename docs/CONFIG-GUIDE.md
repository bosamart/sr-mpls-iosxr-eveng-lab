# Config Guide — The Real Device Configs, Explained Top to Bottom

This walks the **actual shipped configs** ([`../configs/`](../configs/)) block by block, in the
order they appear in the file — so you can open `R1.txt` next to this and understand
every single line.

How it fits with the other docs:

| Doc | What it gives you |
|---|---|
| [`../README.md`](../README.md) | The **build path** — add one feature per phase, verify, repeat. |
| **this file** | The **finished config**, read top to bottom, every block explained. |
| [`PARAMETERS.md`](PARAMETERS.md) | A **knob lookup** — one parameter at a time, with scope tags. |
| [`PE-Template.md`](PE-Template.md) | A **blank template** — fill in the `{{VARIABLES}}` for a new PE. |
| [`CONCEPTS.md`](CONCEPTS.md) | The **theory** — why segment routing works at all. |

There are three roles. Read the **PE** walkthrough first (it has everything); the **P**
and **CE** are short.

- **PE** = R1, R4 — edge routers, hold the customer VRFs and services. *(R4 mirrors R1.)*
- **P** = R2, R3 — pure core, just transport. *(R3 mirrors R2.)*
- **CE** = CE1, CE2 — customer routers, know nothing about MPLS/SR.

Throughout: **domain-wide** values are identical on every node; **per-device** values
change per router (loopback, system-id, SID index, locator, interfaces); **per-service**
values change per VPN (RD, RT, EVI).

---

# Part 1 — PE walkthrough (`R1.txt`)

A PE config has five logical layers, top to bottom in the file:
**(A)** identity & VRFs → **(B)** SR/SRv6 + TE objects → **(C)** EVPN → **(D)** interfaces →
**(E)** IS-IS → **(F)** BGP → **(G)** L2VPN. Build order when pasting is roughly the same.

## A. Hostname, VRFs, and the eBGP policy

```
hostname R1
!
vrf CUST-A
 address-family ipv4 unicast
  import route-target  100:1      ! pull in VPN routes tagged 100:1
  export route-target  100:1      ! tag our routes with 100:1
 !
!
vrf CUST-B                         ! 2nd VRF — exists only for the RD-vs-RT demo
 address-family ipv4 unicast
  import route-target  100:2
  export route-target  100:2
 !
!
route-policy PASS                  ! "permit everything" — required on XR eBGP
  pass
end-policy
```

- **`vrf CUST-A`** is the customer's private routing table. The **route-targets** are the
  membership tag: *export* stamps `100:1` on routes this PE originates; *import* accepts
  any VPN route carrying `100:1`. **RT controls who talks to whom.**
- **`vrf CUST-B`** carries no customer — it's there so you can *see* the RD doing its job:
  the same prefix can live in both VRFs and stay unique because each has a different RD
  (set later under BGP). RT `100:2` keeps it isolated from CUST-A.
- **`route-policy PASS`** is a do-nothing permit. IOS XR applies an implicit *deny* on
  eBGP, so without a policy in **and** out, a CE session comes up but exchanges zero
  routes. This is the single most common "why no prefixes?" gotcha.

## B. Segment Routing — SRv6 locator + SR-TE policy

```
segment-routing
 srv6
  encapsulation
   source-address fcbb:bb00:1::1   ! src IPv6 used when THIS node encaps into SRv6
  !
  locators
   locator MAIN
    micro-segment behavior unode psp-usd   ! uSID format + uN node behavior
    prefix fcbb:bb00:1::/48                ! the /48 all of R1's SIDs are carved from
   !
  !
 !
 traffic-eng
  segment-list SCENIC-R1-TO-R4     ! ordered hop list: R2 → R3 → R4
   index 10 mpls label 16002
   index 20 mpls label 16003
   index 30 mpls label 16004
  !
  policy R1-TO-R4-SCENIC
   color 100 end-point ipv4 4.4.4.4    ! policy identity = (color, endpoint)
   autoroute
    include ipv4 4.4.4.4/32            ! steer 4.4.4.4/32 into this policy (lab method)
   !
   candidate-paths
    preference 100
     explicit segment-list SCENIC-R1-TO-R4
    !
   !
  !
 !
```

- **`srv6 / locator MAIN / prefix fcbb:bb00:1::/48`** — R1's **locator**, a block of IPv6
  from which every SRv6 SID it owns (uN, uA, uDT4, uDX2) is allocated. Per device:
  `fcbb:bb00:1/2/3/4::/48`. `micro-segment behavior unode psp-usd` selects the compressed
  **uSID** format and the **uN** (node) behavior.
- **`encapsulation source-address`** — the IPv6 source R1 stamps on packets it
  encapsulates into SRv6. One per router (`::1`/`::2`/`::3`/`::4`).
- **`traffic-eng`** — the SR-TE policy from Phase 4. The **segment-list** is an ordered
  recipe of prefix-SID labels (R2→R3→R4); the **policy** binds it to a `(color 100,
  endpoint 4.4.4.4)` identity and `autoroute` pulls `4.4.4.4/32` onto it. **This whole
  block lives only on the headend R1** — the P routers need nothing.

## C. EVPN

```
evpn
 segment-routing srv6
  locator MAIN          ! let EVPN carve its uDX2 service SID from MAIN
 !
!
```

One block: it tells EVPN to allocate its L2 service SID (a **uDX2**) from the same
locator. Without it, the Phase-7 pseudowire has no SRv6 SID to use.

## D. Interfaces

```
interface Loopback0
 ipv4 address 1.1.1.1 255.255.255.255      ! the router ID / SR anchor
 ipv6 address fcbb:bb00:1::1/128           ! SRv6 needs a v6 loopback too
!
interface GigabitEthernet0/0/0/0
 vrf CUST-A                                ! customer-facing — lives INSIDE the VRF
 ipv4 address 192.168.11.1 255.255.255.252
!
interface GigabitEthernet0/0/0/1
 ipv4 address 10.12.0.1 255.255.255.252    ! core link to R2
 ipv6 enable                               ! v6 on for SRv6 transport
!
interface GigabitEthernet0/0/0/2
 l2transport                               ! L2 attachment circuit (EVPN-VPWS) — no IP
!
interface GigabitEthernet0/0/0/3
 ipv4 address 10.13.0.1 255.255.255.252    ! core link to R3
 ipv6 enable
!
```

Three kinds of port on a PE:

- **Loopback0** — dual-stacked. The IPv4 `/32` is the SR-MPLS anchor (prefix-SID 16001);
  the IPv6 `/128` is the SRv6 anchor. Everything peers and sources from here.
- **Core links** (Gi0/0/0/1, /0/3) — plain P2P with `ipv6 enable` so both SR-MPLS (v4)
  and SRv6 (v6) can forward across them.
- **Customer ports** — two flavors: the **L3VPN** port (Gi0/0/0/0) is placed `vrf CUST-A`
  and given an IP; the **L2VPN** port (Gi0/0/0/2) is `l2transport` with **no IP** — it
  just feeds raw Ethernet to the pseudowire.

> Note the per-PE asymmetry: on **R4** the L2 port is **Gi0/0/0/0** and the L3 port is
> **Gi0/0/0/1** (opposite of R1). Always check the topology table.

## E. IS-IS — the underlay for both data planes

```
router isis CORE
 is-type level-2-only                  ! flat L2 backbone
 net 49.0001.0000.0000.0001.00         ! area 49.0001 + sys-id ...0001 + NSEL 00
 address-family ipv4 unicast
  metric-style wide                    ! 32-bit metrics — MANDATORY for SR
  segment-routing mpls                 ! SR-MPLS on, claims SRGB 16000–23999
 !
 address-family ipv6 unicast
  metric-style wide
  segment-routing srv6
   locator MAIN                        ! flood the SRv6 locator in IS-IS
  !
 !
 interface Loopback0
  passive                              ! advertise Lo0, no hellos on it
  address-family ipv4 unicast
   prefix-sid index 1                  ! R1=1 … R4=4  → label 16000+index
  !
  address-family ipv6 unicast
  !
 !
 interface GigabitEthernet0/0/0/1
  point-to-point                       ! no DIS/pseudonode, faster adjacency
  address-family ipv4 unicast
   fast-reroute per-prefix             ! TI-LFA: pre-compute a backup...
   fast-reroute per-prefix ti-lfa      ! ...using the topology-independent algorithm
  !
  address-family ipv6 unicast
  !
 !
 interface GigabitEthernet0/0/0/3
  point-to-point
  address-family ipv4 unicast
   fast-reroute per-prefix
   fast-reroute per-prefix ti-lfa
  !
  address-family ipv6 unicast
  !
 !
```

This single process carries **both** transports:

- The **IPv4 AF** runs **SR-MPLS** (`segment-routing mpls`, prefix-SID under the loopback).
- The **IPv6 AF** runs **SRv6** (`segment-routing srv6 / locator MAIN`).
- **`metric-style wide`** is required in *both* AFs — SR sub-TLVs only ride in wide
  metrics.
- Each core interface carries both AFs and the two **TI-LFA** lines (Phase 3). The
  loopback is `passive` and is where the **prefix-sid index** attaches.
- **Per device:** the NET system-id and the prefix-sid index are the only values that
  change (R1=`...0001`/index 1 … R4=`...0004`/index 4). Everything else is domain-wide.

## F. BGP — VPN control plane

```
router bgp 100
 bgp router-id 1.1.1.1
 address-family vpnv4 unicast          ! carry L3VPN routes
 !
 address-family l2vpn evpn             ! carry L2VPN (EVPN) routes
 !
 neighbor 4.4.4.4                       ! the other PE (1.1.1.1 on R4)
  remote-as 100                         ! same AS = iBGP
  update-source Loopback0               ! peer from the loopback
  address-family vpnv4 unicast
   encapsulation-type srv6             ! advertise VPN routes with SRv6 (uDT4), not a label
  !
  address-family l2vpn evpn
  !
 !
 vrf CUST-A
  rd 100:1                              ! Route Distinguisher — makes prefixes unique
  address-family ipv4 unicast
   redistribute connected               ! inject the PE–CE subnet into the VPN
   segment-routing srv6
    locator MAIN
    alloc mode per-vrf                 ! one uDT4 service SID for this VRF
   !
  !
  neighbor 192.168.11.2                  ! the CE (192.168.44.2 on R4)
   remote-as 65001                       ! different AS = eBGP (65002 on R4)
   address-family ipv4 unicast
    route-policy PASS in                 ! REQUIRED on XR eBGP
    route-policy PASS out
   !
  !
 !
 vrf CUST-B                              ! the RD-demo VRF — RD differs (100:2), no CE
  rd 100:2
  address-family ipv4 unicast
   redistribute connected
  !
 !
```

- **Global AFs** `vpnv4 unicast` and `l2vpn evpn` enable the L3 and L2 service families.
- **`neighbor 4.4.4.4`** is the PE↔PE **iBGP** session (same AS 100), sourced from the
  loopback. `encapsulation-type srv6` is what makes the VPN ride **SRv6** (uDT4) instead
  of an MPLS VPN label — flip Phase 5 ↔ Phase 6 with this one line.
- **`vrf CUST-A`**: `rd 100:1` gives uniqueness; `redistribute connected` advertises the
  customer subnet; `segment-routing srv6 / alloc mode per-vrf` allocates the **uDT4**
  service SID. The **`neighbor 192.168.11.2`** block is the PE↔CE **eBGP** session — note
  the mandatory `route-policy PASS`.
- **`vrf CUST-B`** is identical mechanics with `rd 100:2` — the contrast that makes
  RD-vs-RT concrete.

## G. L2VPN — the EVPN-VPWS pseudowire

```
l2vpn
 xconnect group EVPN-VPWS
  p2p CE1-CE2-L2                        ! a point-to-point cross-connect
   interface GigabitEthernet0/0/0/2    ! the local L2 attachment circuit (Gi0/0/0/0 on R4)
   neighbor evpn evi 200 service 1 segment-routing srv6   ! EVPN-signalled PW over SRv6
  !
 !
```

The point-to-point xconnect stitches the local L2 port to a remote endpoint identified by
**EVI 200**, signalled by EVPN, encapsulated over **SRv6** (resolves to the remote PE's
**uDX2** SID). Both PEs must use the same EVI. `service 1` is the local AC identifier.

---

# Part 2 — P walkthrough (`R2.txt`)

A P (core) router is **just transport** — no VRF, no BGP, no services. It only needs to
label-switch and SRv6-forward. That's the whole point of SR: intelligence at the edge,
dumb fast core.

```
hostname R2
!
segment-routing
 srv6
  encapsulation
   source-address fcbb:bb00:2::1
  !
  locators
   locator MAIN
    micro-segment behavior unode psp-usd
    prefix fcbb:bb00:2::/48            ! R2's own locator block
   !
  !
 !
!
interface Loopback0
 ipv4 address 2.2.2.2 255.255.255.255
 ipv6 address fcbb:bb00:2::1/128
!
interface GigabitEthernet0/0/0/0        ! to R1
 ipv4 address 10.12.0.2 255.255.255.252
 ipv6 enable
!
interface GigabitEthernet0/0/0/1        ! to R3 (the cross-link)
 ipv4 address 10.23.0.1 255.255.255.252
 ipv6 enable
!
interface GigabitEthernet0/0/0/3        ! to R4
 ipv4 address 10.24.0.1 255.255.255.252
 ipv6 enable
!
router isis CORE
 is-type level-2-only
 net 49.0001.0000.0000.0002.00          ! sys-id ...0002
 address-family ipv4 unicast
  metric-style wide
  segment-routing mpls
 !
 address-family ipv6 unicast
  metric-style wide
  segment-routing srv6
   locator MAIN
  !
 !
 interface Loopback0
  passive
  address-family ipv4 unicast
   prefix-sid index 2                   ! R2 = index 2 → label 16002
  !
  address-family ipv6 unicast
  !
 !
 interface GigabitEthernet0/0/0/0        ! ...plus /0/1 and /0/3, each with:
  point-to-point
  address-family ipv4 unicast
   fast-reroute per-prefix
   fast-reroute per-prefix ti-lfa
  !
  address-family ipv6 unicast
  !
 !
!
```

Compared with the PE, a P router is **just sections B (locator only), D (interfaces),
and E (IS-IS)** — and even B is only the SRv6 locator, with no TE policy. There is
**no BGP, no VRF, no EVPN, no L2VPN**. It learns every node's prefix-SIDs and locators via
IS-IS and forwards on them. R3 is the same with sys-id `...0003`, index `3`, and its own
link IPs.

> This asymmetry *is* the SR value proposition: adding a new VPN customer touches only the
> two PEs. The core never changes.

---

# Part 3 — CE walkthrough (`CE1.txt`)

The CE is an ordinary router. It has **no idea** SR, MPLS, or VPNs exist — it just runs
eBGP to its PE and has an interface in the shared L2 segment.

```
hostname CE1
!
route-policy PASS                        ! XR eBGP needs it on the CE side too
  pass
end-policy
!
interface Loopback0
 ipv4 address 11.11.11.11 255.255.255.255   ! the "customer prefix" we test reachability to
!
interface GigabitEthernet0/0/0/1
 ipv4 address 192.168.11.2 255.255.255.252  ! L3VPN link up to R1 (the PE)
!
interface GigabitEthernet0/0/0/2
 ipv4 address 172.16.0.1 255.255.255.0      ! L2VPN segment — same subnet as CE2
!
router bgp 65001                          ! customer AS
 bgp router-id 11.11.11.11
 address-family ipv4 unicast
  network 11.11.11.11/32                  ! advertise our loopback to the PE
 !
 neighbor 192.168.11.1                     ! the PE
  remote-as 100                            ! eBGP to provider AS 100
  address-family ipv4 unicast
   route-policy PASS in
   route-policy PASS out
  !
 !
!
```

- **L3VPN side:** eBGP (AS 65001 → AS 100) to the PE on Gi0/0/0/1, advertising its
  loopback with a `network` statement. The PE drops this into VRF CUST-A; the far CE
  reaches it across the SR core.
- **L2VPN side:** Gi0/0/0/2 simply sits in `172.16.0.0/24` — the *same* subnet as CE2's
  L2 port. From the CE's view they're on one Ethernet segment; the EVPN-VPWS pseudowire
  makes that true across the provider core.
- **CE2** mirrors this: AS 65002, loopback `22.22.22.22`, L3 link `192.168.44.2` to R4,
  L2 port in `172.16.0.2/24`.

---

# Part 4 — Build / paste order

If you're pasting the full configs onto blank routers, this order avoids "forward
reference" errors (referring to something not yet defined):

1. **`route-policy PASS`** first — BGP neighbors reference it.
2. **`segment-routing` / `srv6` locator** — IS-IS and BGP reference `locator MAIN`.
3. **`vrf CUST-A` / `CUST-B`** definitions — interfaces and BGP reference them.
4. **Interfaces** — loopback, core links, customer ports.
5. **`router isis CORE`** — needs the interfaces and locator to exist.
6. **`router bgp 100`** — needs VRFs, policy, locator.
7. **`evpn` + `l2vpn`** — needs the locator and the L2 port.
8. **`commit`**, then `show` to verify (see [`../notes/`](../notes/) for the expected output).

On the P routers, only steps 2, 4, 5 apply. On the CEs, only steps 1, 4, 6.

---

# Part 5 — What changes per router (cheat sheet)

| Item | Scope | R1 | R2 | R3 | R4 |
|---|---|---|---|---|---|
| IPv4 Loopback0 | per-device | 1.1.1.1 | 2.2.2.2 | 3.3.3.3 | 4.4.4.4 |
| IPv6 Loopback0 | per-device | fcbb:bb00:1::1 | …:2::1 | …:3::1 | …:4::1 |
| NET system-id | per-device | …0001 | …0002 | …0003 | …0004 |
| prefix-sid index | per-device | 1 | 2 | 3 | 4 |
| SRv6 locator | per-device | fcbb:bb00:1::/48 | …:2::/48 | …:3::/48 | …:4::/48 |
| Role | — | PE | P | P | PE |
| Has BGP / VRF / EVPN? | role | yes | no | no | yes |
| L3 customer port | per-device | Gi0/0/0/0 | — | — | Gi0/0/0/1 |
| L2 customer port | per-device | Gi0/0/0/2 | — | — | Gi0/0/0/0 |

**Domain-wide (identical everywhere):** IS-IS process `CORE`, `is-type level-2-only`,
`metric-style wide`, `segment-routing mpls`, SRGB `16000–23999`, locator name `MAIN`,
uSID behavior `unode psp-usd`, provider AS `100`, RT/RD `100:1` (CUST-A), EVI `200`.

---

*Companion to the phased build in [`../README.md`](../README.md). For one-knob-at-a-time
detail see [`PARAMETERS.md`](PARAMETERS.md); for a blank fill-in template see
[`PE-Template.md`](PE-Template.md).*
