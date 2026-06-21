# CONCEPTS.md

# Segment Routing (SR-MPLS/SRv6) - Conceptual Overview

This document explains the concepts demonstrated in the lab without focusing on vendor-specific commands.

The goal is to understand **why** each phase exists and how all phases connect together.

---

# The Core Idea of Segment Routing

Segment Routing (SR) is a form of source routing.

The ingress router decides how traffic should travel through the network and encodes that path into the packet as an ordered list of segments.

Transit routers do not maintain per-path state.

Their job is simple:

1. Read the top segment.
2. Forward accordingly.
3. Remove the segment when consumed.
4. Repeat until the destination is reached.

Everything in this lab is built from that idea.

---

# Segment Types

## Prefix-SID (Node-SID)

A Prefix-SID identifies a destination router.

Characteristics:

* Globally significant
* Advertised through the IGP
* Follows the shortest path

Examples from this lab:

| Router | Prefix-SID |
| ------ | ---------- |
| R1     | 16001      |
| R2     | 16002      |
| R3     | 16003      |
| R4     | 16004      |

Example:

16004 means:

"Reach router R4."

---

## Adjacency-SID

An Adjacency-SID identifies a specific outgoing link.

Characteristics:

* Locally significant
* Represents an interface
* Used for strict traffic steering

Example:

Instead of:

R1 → R4

an Adjacency-SID can force:

R1 → R2 → R3 → R4

using a specific link.

Think of an Adjacency-SID as:

"Use this exact interface."

---

# Phase 1 - IS-IS Foundation

Objective:

Build reachability between all routers.

IS-IS provides:

* Topology discovery
* Shortest-path calculation
* Loopback reachability

Without a healthy IGP:

* Prefix-SIDs cannot work
* SR-TE cannot work
* TI-LFA cannot work
* L3VPN cannot work

Everything depends on the IGP.

---

# Phase 2 - SR-MPLS

Objective:

Enable Segment Routing using MPLS labels.

Traditional MPLS requires:

* IGP
* LDP

Segment Routing removes LDP.

Instead:

The IGP advertises Prefix-SIDs directly.

Example:

R4 advertises:

Loopback = 4.4.4.4
Prefix-SID = 16004

Every router learns:

16004 = R4

No label distribution protocol is required.

---

# Phase 3 - TI-LFA

Objective:

Provide fast reroute during failures.

Normal path:

R1 → R2 → R4

If a failure occurs:

R1 immediately switches traffic to a precomputed backup path.

Example:

R1 → R3 → R4

The repair path is already programmed before the failure happens.

Result:

Traffic continues forwarding while the IGP reconverges.

This typically provides sub-50ms recovery.

---

# Phase 4 - SR Traffic Engineering

Objective:

Force traffic onto a specific path.

Normal IGP behavior:

R1 chooses the shortest path.

SR-TE behavior:

R1 imposes a Segment List.

Example:

16002
16003
16004

Meaning:

R1 → R2 → R3 → R4

The core does not know about the policy.

Only the ingress router knows.

The path is encoded into the packet itself.

---

# Phase 5 - L3VPN over SR-MPLS

Objective:

Carry customer traffic across the SR core.

Customer:

CE1 ↔ CE2

VPN route:

22.22.22.22/32

The packet uses two labels.

Outer Label:

Transport

Example:

16004

Inner Label:

VPN Service

Example:

VPN Label assigned by BGP

Core routers only examine the transport label.

The VPN information is only relevant at the PE routers.

This separation allows large-scale VPN deployments.

---

# Phase 6 - SRv6

Objective:

Provide Segment Routing using IPv6 instead of MPLS labels.

SR-MPLS:

16004

SRv6:

fcbb:bb00:4::

Both represent:

"Reach R4"

The difference is the encoding.

---

## SRv6 Locator

Example:

fcbb:bb00:4::/48

Locator identifies:

* Router
* Routing domain

Equivalent concept:

Node-SID in SR-MPLS.

---

## SRv6 Service SID

Example:

End.DT4

Function:

1. Receive SRv6 packet
2. Remove SRv6 encapsulation
3. Perform IPv4 lookup in a VRF

Equivalent concept:

VPN Label in MPLS L3VPN.

---

# Relationship Between Phase 5 and Phase 6

A common misconception is that Phase 6 replaces Phase 5.

That is not what happened.

The final lab runs both:

* SR-MPLS
* SRv6

simultaneously.

The customer service remains unchanged.

Only the transport mechanism changes.

Phase 5:

Customer Packet
→ VPN Label
→ Transport Label
→ MPLS Core

Phase 6:

Customer Packet
→ SRv6 SID
→ IPv6 Core

The customer is unaware of the transport change.

---

# The Big Picture

Every phase is the same concept applied differently.

Phase 2:

Segment List = Reach Destination

Phase 3:

Segment List = Backup Path

Phase 4:

Segment List = Traffic Engineering Path

Phase 5:

Segment List = Transport for VPN Service

Phase 6:

Segment List = IPv6-Native Transport and Service

The core principle never changes:

The ingress router decides the path.

The network follows the segments.

That is Segment Routing.
