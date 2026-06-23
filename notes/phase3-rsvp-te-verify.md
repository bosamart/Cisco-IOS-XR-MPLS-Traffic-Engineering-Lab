# Phase 3 — RSVP-TE Basic Tunnel (explicit scenic path): Verification Log

**Date:** 2026-06-23
**Objective:** Signal `tunnel-te1` from R1 to R4 over the **scenic** explicit path
R1→R2→R3→R4 (against the IGP shortest path) using RSVP-TE.

**Result:** ✅ **PASS** — after fixing two config gaps (below), `tunnel-te1` is up/up
on path-option 1, RRO confirms the LSP traverses R1→R2→R3→R4 with 100 Mbps reserved.

---

## Troubleshooting before it came up (two real gaps)

This phase did **not** work on first paste. Two issues, found and fixed in order:

1. **IS-IS MPLS-TE not flooded on R1 and R2.**
   `show mpls traffic-eng link-management interface` showed
   *"Not flooded: No IGP-area has been configured for MPLS TE flooding"* and
   `IGP Neighbor Count: 0` on R1/R2 (R3/R4 were fine). The headend had no TE
   topology, so no path could be computed/validated. **Fix:** re-applied under
   `router isis CORE / address-family ipv4 unicast`:
   `mpls traffic-eng level-2-only` + `mpls traffic-eng router-id Loopback0`, commit.
   After this, `show mpls traffic-eng topology brief` listed all 4 nodes + all links.

2. **`tunnel-te1` had no `path-option`.**
   Even with a complete TE topology, the tunnel stayed down with `Path: not valid`.
   `show mpls traffic-eng tunnels 1 detail` showed no path-option and `Bandwidth
   Requested: 0 kbps` — the tunnel block was only partially pasted. **Fix:**
   re-applied the full `interface tunnel-te1` block (ipv4 unnumbered, destination,
   `path-option 1 explicit name SCENIC-R1-TO-R4`, `path-option 2 dynamic`,
   `signalled-bandwidth 100000`), commit. Tunnel came up immediately.

> **Lesson:** a TE tunnel needs BOTH (a) IGP TE flooding on every router in the
> path, and (b) a valid path-option on the headend. Missing either = tunnel down.

---

## Commands run

```
show rsvp neighbor / show rsvp session            ! per core router
show mpls traffic-eng link-management interface   ! flooding status
show mpls traffic-eng topology brief              ! TE database (R1)
show mpls traffic-eng tunnels brief               ! R1
show mpls traffic-eng tunnels 1 detail            ! R1 — path + RRO
show explicit-paths                               ! R1
```

---

## Tunnel state (R1)

```
RP/0/RP0/CPU0:R1#show mpls traffic-eng tunnels brief
   TUNNEL NAME      DESTINATION   STATUS  STATE
   tunnel-te1       4.4.4.4       up      up
Displayed 1 up, 0 down

RP/0/RP0/CPU0:R1#show explicit-paths
Path SCENIC-R1-TO-R4    status enabled
        10:  next-address strict 10.12.0.2
        20:  next-address strict 10.23.0.2
        30:  next-address strict 10.34.0.2
```

## Tunnel detail — path + reservation (R1)

```
Status:
    Admin: up Oper: up   Path: valid   Signalling: connected
    path option 1, type explicit SCENIC-R1-TO-R4 (Basis for Setup, path weight 30)
      Accumulative metrics: TE 30 IGP 30
    path option 2, type dynamic
    Bandwidth Requested: 100000 kbps  CT0
  Current LSP Info:
    In-use path-option: 1
    Outgoing Interface: GigabitEthernet0/0/0/1, Outgoing Label: 24004
    Router-IDs: local 1.1.1.1   downstream 2.2.2.2
    Path Info:
      Explicit Route:
          Strict, 10.12.0.2     (R2)
          Strict, 10.23.0.2     (R3, via cross-link)
          Strict, 10.34.0.2     (R4)
          Strict, 4.4.4.4
      Session Attributes: Local Prot: Set, Node Prot: Set, BW Prot: Not Set
    Resv Info:
      Record Route:
        IPv4 2.2.2.2 (Node-ID)  Label 24004     <-- R1 -> R2
        IPv4 10.12.0.2          Label 24004
        IPv4 3.3.3.3 (Node-ID)  Label 24005     <-- R2 -> R3 (cross-link)
        IPv4 10.23.0.2          Label 24005
        IPv4 4.4.4.4 (Node-ID)  Label 3         <-- R3 -> R4 (PHP, implicit-null)
        IPv4 10.34.0.2          Label 3
      Tspec: avg rate=100000 kbits
```

---

## Analysis

- **Tunnel up on the scenic path.** Path-option 1 (explicit) is the basis for setup
  with **TE weight 30** (3 hops × 10) — deliberately *longer* than the IGP shortest
  path (weight 20, 2 hops). This is the whole point of TE: forcing a non-shortest path. ✔
- **RRO is the authoritative proof.** The Record Route shows the LSP physically
  traversing **R1 → R2 (2.2.2.2) → R3 (3.3.3.3) → R4 (4.4.4.4)** via the cross-link,
  with label stack 24004 → 24005 → 3 (PHP at R4). ✔
- **Bandwidth reserved:** 100000 kbps (CT0) signalled end to end. ✔
- **FRR requested:** Session Attributes show `Local Prot: Set, Node Prot: Set` —
  the tunnel is asking for node protection (delivered in Phase 5). ✔

### Important: the plain traceroute does NOT use the tunnel yet

```
RP/0/RP0/CPU0:R1#traceroute 4.4.4.4 source 1.1.1.1
 1  10.12.0.2 [MPLS: Label 24005 ...]
 2  10.34.0.2 ... / 10.24.0.2 ...
```

`AutoRoute: disabled` — with no autoroute (and no static route to the tunnel),
nothing in R1's RIB points at `tunnel-te1`, so `traceroute 4.4.4.4` still rides
plain IGP/LDP. The tunnel is up and holding its reserved LSP, but it is **not in
the forwarding path** for 4.4.4.4 until **Phase 4 (autoroute announce)**. For
Phase 3, the tunnel detail / RRO is the correct proof — not the destination traceroute.

### Statefulness (the RSVP-TE teaching point) — optional capture

To show the "every hop holds state" property that distinguishes RSVP-TE from SR-TE,
run on the transit routers while the tunnel is up:

```
show rsvp session        ! on R2 and R3 — PATH/RESV state for the LSP
```

(Before the tunnel was up these were empty; with it up, R2 and R3 each hold RSVP
PATH/RESV state for tunnel-te1. In the SR-TE lab, only the headend holds the path.)

**Conclusion:** RSVP-TE explicit-path tunnel is up and verified on the scenic path.
Proceed to Phase 4 (CSPF + autoroute) — which puts the tunnel into the data path.
