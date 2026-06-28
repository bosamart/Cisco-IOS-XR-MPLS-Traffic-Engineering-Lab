# MPLS Traffic Engineering on Cisco IOS XRv9000 — EVE-NG Lab

![Platform](https://img.shields.io/badge/platform-Cisco%20IOS%20XRv9000-1BA0D7?logo=cisco&logoColor=white)
![IOS XR](https://img.shields.io/badge/IOS%20XR-24.3.1-005073)
![EVE-NG](https://img.shields.io/badge/EVE--NG-Community%20%7C%20Pro-orange)
![Transport](https://img.shields.io/badge/transport-RSVP--TE%20%2B%20LDP-success)
![Phases](https://img.shields.io/badge/phases-9-blueviolet)

A hands-on lab building MPLS Traffic Engineering the **classic way** — using RSVP-TE
for tunnel signaling and LDP for label distribution. This is how operators ran TE
before Segment Routing, and understanding it makes SR-TE's advantages obvious.

> **How to use this lab:**
> Follow the phases one by one. Paste the config, run the verify commands, confirm it
> works — *then* move to the next phase. You don't need to understand everything upfront;
> the concepts click as you build.
>
> **New to MPLS-TE?** Start with the [Beginner Guide](docs/BEGINNER-GUIDE.md) — it
> explains the *why* behind every phase in plain language. Theory reference:
> [`docs/CONCEPTS.md`](docs/CONCEPTS.md) · command cheat-sheet:
> [`docs/TE-Config-Guide.md`](docs/TE-Config-Guide.md).

> **Read this alongside the SR-MPLS lab.**
> Both labs use the **same topology and addressing**. The only difference is the
> transport technology. Comparing them side-by-side is the fastest way to understand
> *why* the industry moved from RSVP-TE to SR-TE.

**GitHub:** https://github.com/bosamart/xrv9000-mpls-te-lab

---

## TL;DR — quick start

For those who already know IOS XR and just want to build it:

1. Build the topology in EVE-NG (6 nodes — wire per the [link table below](#addressing-plan)).
2. Paste each device's full config from [`configs/`](configs/) — `R1.txt`–`R4.txt`, `CE1.txt`, `CE2.txt` are **cumulative** (base Phases 1–8; the Phase 7b and Phase 9 tunnels are throwaway experiments, run from the README, not in the base configs). New XR interfaces may come up admin-down — `no shutdown` when pasting.
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

![MPLS-TE diamond topology](diagrams/topology.svg)

The diamond gives two equal-cost paths (R1→R2→R4 and R1→R3→R4) plus a cross-link
(R2↔R3). This makes TE meaningful — you can force traffic onto a non-shortest path,
and FRR has a real detour to use. The GOLD/BRONZE coloring is the Phase 7 affinity demo.

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
| 7 | Affinity / admin-group coloring | Steer a tunnel by link color, not explicit hops |
| 8 | Auto-bandwidth | Tunnel resizes its own reservation from measured traffic |
| 9 | DS-TE priority + preemption | High-priority tunnel bumps a low-priority one off a full link |

---

## Results

- [x] **Phase 1** — IS-IS adjacencies up, all loopbacks reachable
- [x] **Phase 2** — LDP sessions up, MPLS labels distributed, traceroute shows labels
- [x] **Phase 3** — RSVP-TE tunnel up, explicit scenic path (R1→R2→R3→R4)
- [x] **Phase 4** — CSPF path computed, tunnel in routing table via autoroute
- [x] **Phase 5** — FRR backup tunnel active, link failure reroutes in <50ms
- [x] **Phase 6** — CE1↔CE2 L3VPN ping success over MPLS-TE tunnel
- [x] **Phase 7** — affinity-steered tunnel: GOLD→north path, BRONZE→south path (CSPF, no explicit hops)
- [x] **Phase 8** — auto-bandwidth resized tunnel-te1's reservation (100M → 30M floor) from measured traffic, make-before-break
- [x] **Phase 9** — high-priority tunnel (setup 0) preempted a low-priority one (hold 7) off a full link; victim down with admission/bw PathErr

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

## Phase 7 — Affinity / Admin-Group Coloring

**Objective:** steer a tunnel by **link color** instead of explicit hops. Color the two
disjoint paths, then let CSPF pick by policy — change one line to re-route.

**How it works:** every link gets an *admin-group* (affinity) bit, given a name via an
`affinity-map`. A tunnel's `affinity include <color>` makes CSPF a hard constraint:
only links carrying that color are eligible. This is how operators keep traffic off
(say) high-latency or paid-transit links — color them, then include/exclude by policy.

Here the **north path** (R1–R2–R4) is `GOLD`, the **south path** (R1–R3–R4) is
`BRONZE`, the cross-link is left uncolored. A demo `tunnel-te2` (dynamic path, no
autoroute — leaves `tunnel-te1` untouched) is steered by its affinity.

```
! Every router — affinity-map must be identical network-wide:
mpls traffic-eng
 affinity-map GOLD bit-position 0
 affinity-map BRONZE bit-position 1
 interface <north-link>
  attribute-names GOLD            ! color the link
 interface <south-link>
  attribute-names BRONZE

! Headend R1 — the steered tunnel:
interface tunnel-te2
 ipv4 unnumbered Loopback0
 destination 4.4.4.4
 path-option 1 dynamic
 affinity include GOLD            ! CSPF uses GOLD-only links -> north path
```

**Verify** (tunnel-te2 has no autoroute — read the path from the tunnel detail, not a
destination traceroute):

```
show mpls traffic-eng tunnels 2 detail     ! Explicit Route = 10.12.0.2 -> 10.24.0.2 (north, GOLD)
show mpls traffic-eng topology             ! see Attribute Names / affinity per link

! Re-steer by changing one line:
interface tunnel-te2
 no affinity include GOLD
 affinity include BRONZE                    ! now 10.13.0.2 -> 10.34.0.2 (south, BRONZE)
```

> **Gotchas (all hit during the build):**
> - In IOS-XR a bare `!` is a *comment*, not a submode exit. Use `exit`/`root` to leave
>   `config-mpls-te-if` before configuring `interface tunnel-te2` (it's global, not under
>   `mpls traffic-eng`).
> - Changing affinity on a live tunnel triggers a **make-before-break** reoptimization —
>   the old LSP stays up until the new (re-colored) path installs. No traffic loss.
> - **The default tunnel affinity is `0x0/0xffff` = "use uncolored links only."** The
>   moment you color these links, any tunnel *without* its own affinity (like `tunnel-te1`)
>   can't find an all-uncolored path and goes **down** with `No path ... (affinity)`. Fix:
>   give such tunnels `affinity 0x0 mask 0x0` (ignore colors). This is already in
>   `configs/R1.txt` — see [Phase 7b](#phase-7b--when-explicit-path-and-color-collide-and-verbatim)
>   for the full story.

---

## Phase 7b — When explicit-path and color collide (and `verbatim`)

**Objective:** see what happens when a tunnel is pinned to an explicit path **and**
constrained by affinity at the same time — the path that "looks fine" can refuse to
come up because a link is the wrong color. Then learn the two ways out.

**The key rule:** an explicit-path option is still **CSPF-verified by default** — the
headend checks that *every hop's link* satisfies the tunnel's affinity. If one link
doesn't match, the path-option is invalid and the tunnel goes **down** (unless a
fallback path-option exists). The `verbatim` keyword turns that check **off** — it
signals the listed hops as-is, ignoring the TE database and affinity entirely.

This runs on a **separate `tunnel-te3`** so it can't disturb `tunnel-te1`/L3VPN. It
reuses the existing `SCENIC-R1-TO-R4` explicit path, which crosses R1→R2 (GOLD), the
**uncolored cross-link**, and R3→R4 (BRONZE).

### Step 1 — explicit path + `include GOLD` → tunnel DOWN

```
interface tunnel-te3
 ipv4 unnumbered Loopback0
 destination 4.4.4.4
 path-option 1 explicit name SCENIC-R1-TO-R4
 affinity include GOLD
commit
```

```
show mpls traffic-eng tunnels 3 detail
! Expect: Oper down, Path: not valid. The scenic path crosses the uncolored
! cross-link and a BRONZE link, neither of which is GOLD -> fails verification.
```

The path is perfectly reachable — it's only the **color** that drops it. This is the
"fail-closed" behaviour operators rely on (e.g. an `exclude DRAIN` color tearing a
pinned tunnel off a link being drained for maintenance).

### Step 2 — two ways to bring it back

**(a) `verbatim` — ignore the color, signal the hops anyway:**
```
interface tunnel-te3
 path-option 1 explicit name SCENIC-R1-TO-R4 verbatim
commit
! Expect: up. verbatim skips CSPF/affinity, so the BRONZE/uncolored links are accepted.
```

**(b) Make the path actually comply — pin an all-GOLD path instead:**
```
explicit-path name NORTH-R1-TO-R4
 index 10 next-address strict ipv4 unicast 10.12.0.2   ! R2 (GOLD link)
 index 20 next-address strict ipv4 unicast 10.24.0.2   ! R4 (GOLD link)
!
interface tunnel-te3
 path-option 1 explicit name NORTH-R1-TO-R4    ! no verbatim — color IS checked, and passes
commit
! Expect: up, path R1->R2->R4, because every hop is now on a GOLD link.
```

> **Real-world shape:** you rarely want a hard down, so production tunnels stack
> path-options that degrade gracefully:
> ```
> path-option 1 explicit name PRIMARY              ! pinned + must satisfy affinity
> path-option 2 dynamic                            ! CSPF, still honors affinity
> path-option 3 explicit name LASTRESORT verbatim  ! ignore constraints, just come up
> ```

**Clean up after the experiment:** `no interface tunnel-te3` (and
`no explicit-path name NORTH-R1-TO-R4` if you added it) — `tunnel-te3` is a teaching
tunnel, not part of the base lab.

---

## Phase 8 — Auto-Bandwidth

**Objective:** stop guessing the reservation. Auto-bandwidth measures the tunnel's
actual output rate and, each *application period*, resizes the RSVP reservation to match
— clamped between a min and max — make-before-break.

**Why:** a static `signalled-bandwidth` is either too big (wastes capacity) or too small
(risks congestion). Auto-bw lets the tunnel track real demand: grow toward `max` in the
busy hour, shrink back at night, with no operator touching it.

```
interface tunnel-te1
 signalled-bandwidth 100000          ! starting value; auto-bw takes over from here
 auto-bw
  application 5                      ! resize every 5 min (minimum; default 1440 = 24h)
  bw-limit min 30000 max 500000      ! never reserve below 30M or above 500M
  adjustment-threshold 5             ! only resize if measured rate differs > 5%
```

> `application 5` is a **lab** setting so you can watch it resize in minutes. Production
> uses hours or the 24h default — frequent resizing churns RSVP state across the core.

**Verify** (read these five columns, ignore the rest):

```
show mpls traffic-eng tunnels auto-bw brief
   Tunnel       LSP  Last appl  Requested  Signalled  Highest  Application
   tunnel-te1     5          -     100000     100000        0     2m 31s   <-- before
   tunnel-te1     6      30000      30000      30000        0     4m 39s   <-- after one period
```

- `Highest BW` = peak rate sampled this period · `Requested/Signalled` = live reservation
  · `Last appl BW` = value set at the last resize · `Application Time Left` = countdown.
- The reservation dropped **100000 → 30000** (measured traffic was light, so it
  right-sized down to the `min` floor), and the **LSP ID incremented (5 → 6)** = a
  make-before-break re-signal at the new bandwidth. No outage.
- In the detail, `Bandwidth: 100000` is the *configured* base; `Bandwidth Requested:
  30000` is what auto-bw *actually* reserves now.

> A virtual lab can't push >30 Mbps, so you'll see it shrink to the floor rather than
> grow. The mechanism is identical either way — measure, then resize within bounds.

**Revert to static if desired:** `interface tunnel-te1 / no auto-bw`.

---

## Phase 9 — DS-TE Priority + Preemption

**Objective:** show the *one* case where bandwidth makes a tunnel go **down** — a
higher-priority tunnel preempting a lower-priority one off a link that's out of
reservable bandwidth.

**The model:** every tunnel has a **setup** and **hold** priority, `0` (best) to `7`
(worst), default `7 7`. Setup = how aggressively it preempts others coming up; hold = how
well it resists being preempted. **Rule:** a new tunnel preempts an existing one when
`new setup < existing hold` (lower number wins).

This runs on two throwaway tunnels forced over R1→R3→R4, each wanting 600M on the 1G link —
only one fits. (Both need `affinity 0x0 mask 0x0` — the south links are BRONZE, and the
default `0x0/0xffff` affinity would reject them; the same Phase 7b trap bites new tunnels.)

```
! Low-priority tunnel:
interface tunnel-te5
 ipv4 unnumbered Loopback0
 destination 4.4.4.4
 priority 7 7
 affinity 0x0 mask 0x0
 signalled-bandwidth 600000
 path-option 1 explicit name SOUTH-R1-TO-R4    ! 10.13.0.2 -> 10.34.0.2

! High-priority tunnel (bring up second):
interface tunnel-te6
 priority 0 0
 affinity 0x0 mask 0x0
 signalled-bandwidth 600000
 path-option 1 explicit name SOUTH-R1-TO-R4
```

**Verify — watch the bandwidth change hands:**

```
show mpls traffic-eng link-management bandwidth-allocation     ! the proof
```
- With only te5 up: `Gi0/0/0/3` shows **600000 locked at PRIORITY 7**.
- After te6 comes up: **te5 is DOWN**, te6 is up, and the **600000 is now locked at
  PRIORITY 0**. The reservation moved from the 7-row to the 0-row.

```
show mpls traffic-eng tunnels 5 detail
! Last Signalled Error: PathErr(1,2) admission (1), bandwidth unavailable (2)
! -> the preemption signature; then "No path (bw)" because te5 has no fallback.
```

**The lesson (your bandwidth question, fully answered):** over-sending a tunnel drops
nothing — a reservation is admission-control bookkeeping, not a policer. The *only* way
bandwidth takes a tunnel down is **preemption**: when the books are full, a higher-priority
tunnel bumps a lower-priority one (`0 < 7`). Give critical services low priority numbers so
they win, and give best-effort tunnels a dynamic fallback so being preempted means
*reroute*, not outage. Default preemption is **hard** (instant tear); `soft-preemption`
reroutes the victim gracefully first.

**Cleanup:** `no interface tunnel-te5` / `no interface tunnel-te6` /
`no explicit-path name SOUTH-R1-TO-R4`, and restore `rsvp interface Gi0/0/0/3 / bandwidth
1000000`.

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

## Going further — inter-area TE & PCE

This lab is a single IS-IS Level-2 area, so it doesn't cross area boundaries. In a real
operator (access / aggregation / core in separate areas) the headend can't see past its own
area, and TE has to be solved with **loose-hop expansion** (classic) or a **PCE + BGP-LS**
(modern, SR-friendly). That's the natural next concept — written up in
[`docs/INTER-AREA-TE-and-PCE.md`](docs/INTER-AREA-TE-and-PCE.md).

---

## References

- Cisco IOS XR MPLS Traffic Engineering Configuration Guide
- IETF RFC 3209 (RSVP-TE Extensions for LSP Tunnels)
- IETF RFC 4090 (RSVP-TE FRR)
- EVE-NG documentation — adding Cisco IOS XRv9000

---

*Lab designed by BO SAM ATH. Compare with SR-MPLS lab for full transport evolution picture.*
