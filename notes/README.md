# SR-MPLS Lab Notes

## Lab Goals
- Understand Segment Routing with MPLS data plane
- Build and test SR-MPLS topologies

## Topics Covered
- [x] SR-MPLS basics: Node SID, Adjacency SID
- [x] IS-IS / OSPF SR extensions
- [x] SR Traffic Engineering (SR-TE)
- [x] TI-LFA Fast Reroute
- [x] L3VPN over SR-MPLS and SRv6
- [x] EVPN-VPWS (L2VPN) over SR-MPLS and SRv6

## Lab Topology
> Topology diagram: 4-node diamond (R1-PE, R2-P, R3-P, R4-PE) + CE1, CE2
> See README.md in this lab for full topology

## Per-Phase Verification Logs
Real `show`-command captures + analysis for each phase:

- [Phase 1 — IS-IS underlay](phase1-isis-verify.md) ✅
- [Phase 2 — SR-MPLS prefix-SIDs](phase2-srmpls-verify.md) ✅
- [Phase 3 — TI-LFA fast reroute](phase3-tilfa-verify.md) ✅
- [Phase 4 — SR-TE explicit path](phase4-srte-verify.md) ✅
- [Phase 5 — L3VPN over SR-MPLS](phase5-l3vpn-verify.md) ✅
- [Phase 6 — L3VPN over SRv6 (uDT4)](phase6-srv6-verify.md) ✅
- [Phase 7 — EVPN-VPWS over SRv6 (uDX2)](phase7-evpn-vpws-verify.md) ✅
- [Phase 8 — SR-PCE + ODN (controller-driven SR-TE)](phase8-srpce-odn-verify.md) ✅ *(extension)*

## Key Configs
> Device configs stored in `../configs/`

## GitHub Repo
https://github.com/bosamart/sr-mpls-iosxr-eveng-lab

## Session Log
| Date | Topic | Outcome |
|------|-------|---------|
| 2026-06-24 | Full 7-phase lab: IS-IS, SR-MPLS, TI-LFA, SR-TE, L3VPN, SRv6, EVPN-VPWS | All phases complete ✅ |
| 2026-06-27 | Phase 8 extension: SR-PCE + ODN (BGP-LS, PCEP, color-100 trigger); light CSR CE3/CE4 sites | ODN verified ✅ (affinity steering not exposed under ODN on this XRv9000 build) |
