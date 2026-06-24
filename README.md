# MPLS Traffic Engineering on Cisco IOS XRv9000 — EVE-NG Lab

![Platform](https://img.shields.io/badge/platform-Cisco%20IOS%20XRv9000-1BA0D7?logo=cisco&logoColor=white)
![IOS XR](https://img.shields.io/badge/IOS%20XR-24.3.1-005073)
![EVE-NG](https://img.shields.io/badge/EVE--NG-Community%20%7C%20Pro-orange)
![Transport](https://img.shields.io/badge/transport-RSVP--TE%20%2B%20LDP-success)
![Phases](https://img.shields.io/badge/phases-6-blueviolet)

A hands-on lab building MPLS Traffic Engineering the **classic way** — using RSVP-TE
for tunnel signaling and LDP for label distribution. This is how operators ran TE
before Segment Routing, and understanding it makes SR-TE's advantages obvious.

> **How to use this lab:**
> Follow the phases one by one. Paste the config, run the verify commands, confirm it
> works — *then* move to the next phase. You don't need to understand everything upfront;
> the concepts click as you build.
>
> Want the theory first? → [`docs/CONCEPTS.md`](docs/CONCEPTS.md)

> **Read this alongside the SR-MPLS lab.**
> Both labs use the **same topology and addressing**. The only difference is the
> transport technology. Comparing them side-by-side is the fastest way to understand
> *why* the industry moved from RSVP-TE to SR-TE.

**GitHub:** https://github.com/bosamart/xrv9000-mpls-te-lab

---

## TL;DR — quick start

For those who already know IOS XR and just want to build it:

1. Build the topology in EVE-NG (6 nodes — wire per the [link table below](#addressing-plan)).
2. Paste each device's full config from [`configs/`](configs/) — `R1.txt`–`R4.txt`, `CE1.txt`, `CE2.txt` are **cumulative** (all 6 phases). New XR interfaces may come up admin-down — `no shutdown` when pasting.
3. Verify end to end:

```
! on R1 — TE tunnel up and in the routing table
show mpls traffic-eng tunnels brief
show route 4.4.4.4/32

! on R1 — transport reachable over the tunnel
ping 4.4.4.4 source Loopback0

! on CE1 — L3VPN works across the TE core
ping 22.22.22.22 source 11.11.11.11
```

All `!!!!!` → the whole stack (IS-IS → LDP → RSVP-TE → CSPF/autoroute → FRR → L3VPN) is up. Want to learn instead of speed-run? Skip this and follow the phases below.

---

## Lab Environment

| Component | Detail |
|---|---|
| Emulator | EVE-NG (Community/Pro) |
| Node image | Cisco IOS XRv9000 (24.3.1) |
| Core/PE nodes | 4 × XRv9000 (R1–R4) |
| CE nodes | 2 × Cisco IOS XRv9000 (CE1, CE2) |
| IGP | IS-IS Level-2-only |
| Label distribution | LDP (Phase 2) + RSVP-TE (Phase 3+) |

---

## Topology

Same diamond as the SR-MPLS lab — same routers, same links, same addresses.

```
             +-------+
    +--------|  R2   |--------+
    |        |  (P)  |        |
    |        +---+---+        |
    |            | cross-link |
+---+----+       |         +--+-----+
|  R1    |       |         |  R4    |
|  (PE)  |       |         |  (PE)  |
+---+----+       |         +--+-----+
    |        +---+---+        |
    +--------|  R3   |--------+
    |        |  (P)  |        |
    |        +-------+        |
 +--+--+                   +--+--+
 | CE1 |                   | CE2 |
 +-----+                   +-----+
```

The diamond gives two equal-cost paths (R1→R2→R4 and R1→R3→R4) plus a cross-link
(R2↔R3). This makes TE meaningful — you can force traffic onto a non-shortest path,
and FRR has a real detour to use.

---

## Addressing Plan

| Node | Role | Loopback0 |
|------|------|-----------|
| R1 | PE | 1.1.1.1/32 |
| R2 | P  | 2.2.2.2/32 |
| R3 | P  | 3.3.3.3/32 |
| R4 | PE | 4.4.4.4/32 |

| Link | Subnet | A-end | B-end |
|------|--------|-------|-------|
| R1–R2 | 10.12.0.0/30 | R1 Gi0/0/0/1 .1 | R2 Gi0/0/0/0 .2 |
| R1–R3 | 10.13.0.0/30 | R1 Gi0/0/0/3 .1 | R3 Gi0/0/0/2 .2 |
| R2–R3 (cross) | 10.23.0.0/30 | R2 Gi0/0/0/1 .1 | R3 Gi0/0/0/0 .2 |
| R2–R4 | 10.24.0.0/30 | R2 Gi0/0/0/3 .1 | R4 Gi0/0/0/2 .2 |
| R3–R4 | 10.34.0.0/30 | R3 Gi0/0/0/4 .1 | R4 Gi0/0/0/3 .2 |
| CE1–R1 | 192.168.11.0/30 | CE1 Gi0/0/0/1 .2 | R1 Gi0/0/0/0 .1 |
| CE2–R4 | 192.168.44.0/30 | CE2 Gi0/0/0/0 .2 | R4 Gi0/0/0/1 .1 |

**L3VPN:** VRF `CUST-A`, RD/RT `100:1`. CE1 AS 65001 Lo0 `11.11.11.11`. CE2 AS 65002 Lo0 `22.22.22.22`. Provider AS 100.

---

## Phases

| Phase | Topic | Key concept |
|-------|-------|-------------|
| 1 | IS-IS baseline | Same IGP foundation as SR lab |
| 2 | LDP | Label distribution without TE — the "plain MPLS" state |
| 3 | RSVP-TE — basic tunnel | Signaled explicit path; core becomes stateful |
| 4 | MPLS-TE — CSPF + autoroute | Headend computes constrained path; routes via tunnel |
| 5 | FRR (Fast Reroute) | Link/node protection — compare to TI-LFA in SR lab |
| 6 | L3VPN over MPLS-TE | Same service as SR lab, different transport |

---

## Results

- [x] **Phase 1** — IS-IS adjacencies up, all loopbacks reachable
- [x] **Phase 2** — LDP sessions up, MPLS labels distributed, traceroute shows labels
- [x] **Phase 3** — RSVP-TE tunnel up, explicit scenic path (R1→R2→R3→R4)
- [x] **Phase 4** — CSPF path computed, tunnel in routing table via autoroute
- [x] **Phase 5** — FRR backup tunnel active, link failure reroutes in <50ms
- [x] **Phase 6** — CE1↔CE2 L3VPN ping success over MPLS-TE tunnel

---

## Phase 1 — IS-IS baseline

**Objective:** every loopback reachable via IS-IS L2 before any MPLS.

```
show isis neighbors
show route isis
ping 4.4.4.4 source 1.1.1.1
```

Expected: full adjacencies, ECMP routes to R4 via both R2 and R3.

---

## Phase 2 — LDP

**Objective:** enable plain MPLS forwarding with LDP distributing labels.

**Why LDP first:** before RSVP-TE you need basic label switching working.
LDP assigns a label to every IGP prefix and distributes it to neighbors.
No path control — just shortest-path MPLS forwarding.

```
mpls ldp
 router-id 1.1.1.1
 interface GigabitEthernet0/0/0/1
 interface GigabitEthernet0/0/0/3
```

**Verify**

```
show mpls ldp neighbor              ! LDP sessions up
show mpls ldp bindings              ! labels allocated per prefix
show mpls forwarding                ! MPLS FIB populated
traceroute 4.4.4.4 source 1.1.1.1  ! shows MPLS label in transit
```

Expected: traceroute shows a label pushed at R1, popped at R3 (PHP), plain IP at R4.

---

## Phase 3 — RSVP-TE: basic tunnel with explicit path

**Objective:** signal a TE tunnel that takes the scenic path R1→R2→R3→R4 — against the
IGP shortest path — using RSVP.

**Why RSVP-TE:** RSVP reserves bandwidth *hop by hop* along the path. Each router on
the path installs RSVP state (the "PATH" and "RESV" messages). This is why RSVP-TE is
called "stateful" — unlike SR-TE where only the headend holds the path.

Three things to enable on every core router:
1. RSVP on the interface (with bandwidth pool)
2. `mpls traffic-eng` on the interface
3. IS-IS `mpls traffic-eng` extensions (to flood TE topology)

On the headend (R1 only):
4. `explicit-path` — the hop-by-hop addresses
5. `interface tunnel-te1` — the TE tunnel itself

```
! Every core router:
rsvp
 interface GigabitEthernet0/0/0/1
  bandwidth 1000000
!
mpls traffic-eng
 interface GigabitEthernet0/0/0/1
!
router isis CORE
 address-family ipv4 unicast
  mpls traffic-eng level-2-only
  mpls traffic-eng router-id Loopback0

! R1 headend only:
explicit-path name SCENIC-R1-TO-R4
 index 10 next-address strict ipv4 unicast 10.12.0.2
 index 20 next-address strict ipv4 unicast 10.23.0.2
 index 30 next-address strict ipv4 unicast 10.34.0.2

interface tunnel-te1
 ipv4 unnumbered Loopback0
 destination 4.4.4.4
 path-option 1 explicit name SCENIC-R1-TO-R4
 path-option 2 dynamic
```

**Verify**

```
show rsvp neighbor                          ! RSVP sessions on each link
show mpls traffic-eng tunnels               ! tunnel-te1 up, admin/oper state
show mpls traffic-eng tunnels detail        ! path taken, bandwidth reserved
traceroute tunnel-te1                       ! 3 hops: R2→R3→R4
```

> **Understand:** `show rsvp session` on R2 and R3 will show RSVP PATH/RESV state for
> the tunnel. This is the "statefulness" — every transit router knows about this tunnel.
> Compare: in the SR-TE lab, only R1 knows about the policy. R2 and R3 just forward labels.

---

## Phase 4 — MPLS-TE: CSPF and autoroute

**Objective:** let the headend compute the path automatically using CSPF, and inject
the tunnel into the routing table so traffic follows it.

**CSPF (Constrained Shortest Path First):** the headend runs its own SPF on the TE
topology (flooded by IS-IS TE extensions), factoring in bandwidth constraints and
admin-groups. The result is a path-option computed automatically — no explicit-path needed.

**Autoroute announce:** without this, the tunnel exists but nothing uses it. Autoroute
makes the tunnel appear as a next-hop in the routing table for destinations behind it.

```
interface tunnel-te1
 autoroute announce
 signalled-bandwidth 100000      ! reserve 100 Mbps
 path-option 1 dynamic           ! CSPF computes the path
```

**Verify**

```
show mpls traffic-eng tunnels detail         ! CSPF path chosen, BW reserved
show route 4.4.4.4/32 detail                ! next-hop = tunnel-te1
show cef 4.4.4.4/32 detail                  ! tunnel as forwarding path
show mpls traffic-eng link-management bandwidth-allocation
```

> **Understand:** `show mpls traffic-eng topology` shows the TE database — bandwidth
> available on each link. CSPF reads this to find the best path meeting the constraint.
> When you reserve bandwidth on the tunnel, that bandwidth is subtracted from the link's
> available pool in every router's TE topology database.

---

## Phase 5 — FRR (Fast Reroute)

**Objective:** protect the TE tunnel against a link or node failure with a pre-computed
bypass tunnel, rerouting in <50ms.

**How RSVP-TE FRR works:** the Point of Local Repair (PLR — the router upstream of the
failure) detects the failure and immediately switches traffic onto a **bypass tunnel**
that was pre-signaled around the protected link or node. The bypass tunnel is itself an
RSVP-TE tunnel.

**Comparison to TI-LFA (SR lab):** TI-LFA is automatic — IS-IS computes the backup
path and SR labels encode it. RSVP FRR requires you to explicitly configure backup
tunnels (or enable auto-tunnel backup). TI-LFA requires no core state; RSVP FRR
adds more RSVP state.

**Auto-tunnel backup** (simplest approach in IOS-XR). Two gotchas the lab verification
surfaced — both are now in the configs:

1. Auto-created tunnels need a **source address**, or every bypass stays down with
   *"No IP source address is configured"*:

   ```
   ! global, on each PLR:
   ipv4 unnumbered mpls traffic-eng Loopback0
   ```

2. Protection is enabled **per on-path interface**. The global block only sets the
   tunnel-id pool — you must also enable it on the interface the LSP egresses:

   ```
   ! On the PLRs (the P routers):
   mpls traffic-eng
    interface GigabitEthernet0/0/0/1   ! R2: the ON-PATH cross-link to R3
     auto-tunnel backup
    !
    auto-tunnel backup
     tunnel-id min 100 max 200
   ```

```
! Enable FRR on the protected tunnel (headend R1):
interface tunnel-te1
 fast-reroute protect node
```

> **Node protection only works at R2 in this topology.** At R3 the next hop is R4 —
> the tunnel tail — so there is no next-next hop to bypass to; R3 can only do link
> protection. R2 has a direct link to R4, so it can bypass node R3 entirely.

**Verify**

```
show mpls traffic-eng tunnels backup         ! auto-bypass tunnels (want Oper: up)
show mpls traffic-eng fast-reroute database  ! protected LSP -> bypass, state Ready

! Prove it: long ping, then shut the ON-PATH cross-link mid-ping
ping 4.4.4.4 source 1.1.1.1 count 100000
! (on R2) interface GigabitEthernet0/0/0/1 -> shutdown   ! NOT Gi0/0/0/3 (off-path!)
! FRR database flips Ready -> Active; expect near-zero loss
```

---

## Phase 6 — L3VPN over MPLS-TE

**Objective:** carry VRF `CUST-A` (CE1↔CE2) over the TE tunnel, proving TE is the
transport for a real service.

**How it works:** MP-BGP VPNv4 is unchanged from the SR lab (Phase 5). The difference
is that the VPN label's transport (the outer label) now rides the TE tunnel instead of
an LDP LSP. When autoroute is enabled on tunnel-te1, BGP uses the tunnel as the
next-hop path to R4 — the VPN label stack sits inside the TE tunnel.

```
! Same VRF/BGP config as SR-MPLS lab Phase 5 — no changes needed
! Autoroute on tunnel-te1 handles the transport automatically
```

**Verify**

```
show bgp vpnv4 unicast summary
show route vrf CUST-A
show cef vrf CUST-A 22.22.22.22 detail   ! resolves via tunnel-te1
! from CE1:
ping 22.22.22.22 source 11.11.11.11
```

> **Key insight:** the service config (VRF, RD/RT, PE-CE BGP) is *identical* to the
> SR-MPLS lab. Only the transport changed — from SR prefix-SID to RSVP-TE tunnel.
> This is why the industry says services are decoupled from transport.

---

## SR-MPLS vs RSVP-TE — Quick Comparison

| Feature | RSVP-TE (this lab) | SR-TE (SR-MPLS lab) |
|---|---|---|
| Path signaling | RSVP PATH/RESV messages | None — headend pushes label stack |
| Core state | Every hop stores tunnel state | Only headend holds policy |
| Fast reroute | Pre-signaled bypass tunnels | TI-LFA (automatic, no bypass tunnel) |
| Path encoding | IP hop addresses (explicit-path) | MPLS label stack (segment-list) |
| Scalability | O(tunnels × hops) state in core | O(tunnels) state at headend only |
| Migration | Existing MPLS networks | New builds or MPLS migration |

---

## References

- Cisco IOS XR MPLS Traffic Engineering Configuration Guide
- IETF RFC 3209 (RSVP-TE Extensions for LSP Tunnels)
- IETF RFC 4090 (RSVP-TE FRR)
- EVE-NG documentation — adding Cisco IOS XRv9000

---

*Lab designed by BO SAM ATH. Compare with SR-MPLS lab for full transport evolution picture.*
