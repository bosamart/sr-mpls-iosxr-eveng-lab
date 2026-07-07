# Packet Captures — the data-plane proof

The `notes/` verify logs capture the **control plane** (`show` output — what each router *intends*).
This folder is the other half: the **data plane** — the actual bytes on the wire. Together they prove
a phase both *decided* correctly and *forwarded* correctly.

Every phase in the [README](../README.md) has a **🔬 On the wire** callout telling you which link to
tap, the Wireshark filter, and what to look for. Drop the resulting `.pcapng` here.

---

## How to capture in EVE-NG

1. Start the lab and get the phase working (its `show` verify passes).
2. In the EVE-NG topology, **right-click the link** you want to tap → **Capture** → pick the
   interface. EVE-NG streams that link into **Wireshark** on your client (needs the EVE-NG client
   integration pack / Wireshark installed locally).
3. Generate traffic (the `ping` / `traceroute` in the callout), stop the capture, and **Save As**
   `captures/phaseN-<short-desc>.pcapng`.

> Tip: capture *before* you start the ping so you catch the first packets. For the TI-LFA test
> (Phase 3), start the capture and the ping, then shut the link — the interesting frames are the
> ones right after the failure.

---

## What to capture, per phase

| Phase | Link to tap | Filter | The one thing to see |
|-------|-------------|--------|----------------------|
| 1 · IS-IS | R1–R2 | `isis` | IS-IS PDUs directly on L2 (no IP header); LSP TLVs |
| 2 · SR-MPLS | R1–R2 | `mpls` | label `16004` on data; Prefix-SID sub-TLV in the LSP |
| 3 · TI-LFA | R1–R3 (backup) | `mpls` | repair stack appears on the backup link at failover |
| 4 · SR-TE | R1–R2, then each hop | `mpls` | stack `{16002,16003,16004}` shrinking one label per hop |
| 5 · L3VPN (MPLS) | R1–R2 | `mpls` | **two-label** stack (transport SID + VPN label) |
| 6 · L3VPN (SRv6) | R1–R2 | `ipv6` | native IPv6 to the **uDT4** SID — no MPLS |
| 7 · EVPN-VPWS | R1–R2 | `ipv6` | Ethernet frame inside IPv6 to the **uDX2** SID |

Naming: `phase1-isis.pcapng`, `phase5-l3vpn-2label.pcapng`, etc. Keep them small — a few pings is
plenty.

---

## Reading them in Wireshark

- **MPLS labels** show as *MultiProtocol Label Switching Header* → `Label: 16004`. A two-label stack
  lists two headers; the bottom one has `Bottom of Label Stack: 1`.
- **SRv6**: expand the outer *Internet Protocol Version 6* header — the **Destination** address *is*
  the SID (`fcbb:bb00:4:e0xx::`). If a Segment Routing Header is present it appears as *Routing
  Header for IPv6 (Segment Routing)*; with a single reduced SID it may be absent (H.Encaps.Red).
- **IS-IS / BGP / RSVP** dissect automatically in any modern Wireshark (≥ 3.x). For BGP, filter
  `bgp` and expand *UPDATE Message* → *Path Attributes* to read AS_PATH, communities, MP_REACH, RD.

---

## Why this is worth doing

For a learner, the capture is the moment the abstraction becomes real — you *see* the label stack or
the SRH instead of trusting a `show`. For a portfolio, annotated captures are rare and read
immediately as data-plane depth: most lab repos stop at `show` output. A screenshot of the
`{16002,16003,16004}` stack shrinking hop by hop, or the same L3VPN packet as MPLS (Phase 5) vs SRv6
(Phase 6), tells the whole SR story in two frames.
