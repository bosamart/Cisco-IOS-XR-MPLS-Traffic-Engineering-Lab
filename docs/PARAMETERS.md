# Parameter Reference — MPLS-TE Knobs Explained

Same format as the SR-MPLS lab. Every parameter tagged with scope:
- **[DOMAIN-WIDE]** — must match on every node
- **[PER-DEVICE]** — unique per router
- **[PER-TUNNEL]** — set on the headend per tunnel
- **[LOCAL]** — only meaningful on that router

---

## IS-IS TE extensions

### `mpls traffic-eng level-2-only` under IS-IS address-family — [DOMAIN-WIDE]
**Job:** tells IS-IS to flood TE link attributes (available bandwidth, admin-groups,
TE metric) in its LSPs. Every router gets a complete TE topology database.
**If missing:** CSPF has no bandwidth data — it falls back to hop-count only.
**Must match?** Enable on all routers. The level must match your IS-IS level.

### `mpls traffic-eng router-id Loopback0` under IS-IS — [PER-DEVICE]
**Job:** identifies this router in the TE topology. Tunnels are addressed to this ID.
**If wrong:** tunnel destination won't resolve in CSPF topology.
**Must match?** Must match the loopback used as tunnel destination on remote PE.

---

## RSVP

### `bandwidth 1000000` under `rsvp interface` — [PER-DEVICE] (per interface)
**Job:** the total RSVP-reservable bandwidth pool on this interface, in kbps.
CSPF checks this pool when computing paths.
**If too low:** tunnels requesting more bandwidth than available will fail CSPF.
**If too high:** you can over-commit — reserve more than the physical link has.
**Must match?** No, each interface sets its own pool. Set it to the physical line rate
or a percentage of it (some operators reserve 80% to leave headroom for non-TE traffic).

---

## MPLS-TE global

### `mpls traffic-eng interface GigXXX` — [PER-DEVICE]
**Job:** enables MPLS-TE on this interface (allows it to carry TE tunnels).
**If missing on a transit router:** the explicit-path or CSPF will not route through it.
**Must match?** All interfaces that carry TE traffic must have this.

### `reoptimize 300` — [LOCAL]
**Job:** every 300 seconds, the headend checks if a better path exists for its tunnels
(more bandwidth, better metric) and re-signals if so.
**If missing:** tunnels stay on their current path even if a better one opens up.

### `auto-tunnel backup tunnel-id min 100 max 200` (global) — [PER-DEVICE] (on PLRs)
**Job:** sets the tunnel-id *pool* used for auto-created RSVP bypass tunnels.
**Important:** the global line alone does **not** protect anything — protection is enabled
**per interface** with `mpls traffic-eng interface X / auto-tunnel backup` on the on-path
egress link. (Lab lesson: a per-interface line on the *wrong* interface protects nothing.)
**If missing:** FRR won't work even with `fast-reroute` on the tunnel — no bypass exists.
**Must match?** Enable on every PLR. The tunnel-id range must not overlap manual tunnels.

### `ipv4 unnumbered mpls traffic-eng Loopback0` (global) — [PER-DEVICE] (on PLRs)
**Job:** gives **auto-created** TE tunnels (bypass backups) their source IP address.
**If missing:** every auto-tunnel backup stays `Oper: down` with
*"No IP source address is configured"* (`Src: 0.0.0.0`) — and FRR silently never arms.
Manual `tunnel-te` interfaces don't need it (they have `ipv4 unnumbered` on the interface);
auto-tunnels need this global form. **Required on any router running `auto-tunnel backup`.**

---

## Affinity / admin-group coloring (Phase 7)

### `affinity-map NAME bit-position N` under `mpls traffic-eng` — [DOMAIN-WIDE]
**Job:** maps a human name (GOLD) to an admin-group bit (0). Tunnels reference the name.
**Must match?** **Yes — identical on every router**, or a color means different things in
different places. (GOLD=bit 0, BRONZE=bit 1 here.)

### `attribute-names NAME` under `mpls traffic-eng interface X` — [PER-DEVICE] (per link)
**Job:** paints a link with one or more affinity colors. Color **both ends** of a link
consistently. Uncolored = no admin-group bits set.
**If wrong:** the link is considered the wrong color and tunnels include/exclude it wrongly.

### `affinity include NAME` / `affinity exclude NAME` — [PER-TUNNEL] (named model)
**Job:** constrains CSPF to links that carry (include) / don't carry (exclude) the color.
`include` makes color a hard constraint on every hop.

### `affinity 0x0 mask 0x0` — [PER-TUNNEL] (legacy bitmap; "ignore colors")
**Job:** mask = which bits to check; `mask 0x0` = check none = accept any link.
**Why you need it:** the **default** tunnel affinity is `0x0/0xffff` = *"uncolored links
only."* Once links are colored, a tunnel without its own affinity goes down with
`No path (affinity)`. Set `affinity 0x0 mask 0x0` so it rides colored links too.

---

## Explicit path

### `explicit-path name X` / `index N next-address strict ipv4 unicast A.B.C.D` — [PER-TUNNEL] (headend)
**Job:** defines the exact hop sequence for a tunnel. `strict` means the packet must
go directly to this next-hop (not via other hops). `loose` allows the IGP to fill in
intermediate hops.
**If wrong address:** RSVP PATH setup fails at the hop where the address doesn't match
an interface. The tunnel stays down.
**Strict vs loose:** use strict for explicit TE paths; loose when you only want to
constrain part of the path and let CSPF fill the rest.

---

## tunnel-te interface

### `destination A.B.C.D` — [PER-TUNNEL]
**Job:** the tunnel tailend — the router-id of the egress PE (must match its `mpls
traffic-eng router-id`).
**If wrong:** tunnel can't be established.

### `path-option 1 explicit name X` / `path-option 2 dynamic` — [PER-TUNNEL]
**Job:** ordered list of path choices. Preference 1 is tried first; if it fails, the
next is tried.
- `explicit`: use the named explicit-path
- `dynamic`: CSPF computes the path automatically
**Best practice:** always add `path-option 2 dynamic` as a fallback. If the explicit
path fails (link down, no bandwidth), the tunnel falls back to dynamic instead of going
down entirely.

### `signalled-bandwidth 100000` — [PER-TUNNEL] (kbps)
**Job:** bandwidth RSVP reserves along the path. CSPF checks that every link has
≥ this much available.
**If too high:** CSPF can't find a path with enough bandwidth → tunnel stays down.
**If 0 / not set:** no bandwidth is reserved → tunnel comes up but no admission control.

### `autoroute announce` — [PER-TUNNEL] (headend)
**Job:** injects the tunnel as a next-hop into the routing table for the tailend's
loopback and any prefixes behind it.
**If missing:** the tunnel is up but unused — traffic still follows IGP shortest path.

### `fast-reroute` — [PER-TUNNEL] (headend)
**Job:** enables FRR on this tunnel. When the PLR detects a failure, it switches to
the bypass tunnel.
**Requires:** bypass tunnels (via `auto-tunnel backup` on PLRs) to exist first.

### `fast-reroute protect bandwidth` / `protect node` — [PER-TUNNEL]
**Job:** what type of protection to request.
- `protect bandwidth`: bypass must have enough bandwidth for this tunnel
- `protect node`: use NNHOP bypass for node protection, not just link protection
**Note:** node protection needs a next-next hop to bypass to — impossible where the next
hop is the tail (the PLR can only do link protection there).

### `path-option N explicit name X verbatim` — [PER-TUNNEL]
**Job:** the `verbatim` keyword signals the explicit hops **as-is**, skipping CSPF and
affinity/bandwidth verification. Use it to force a path regardless of constraints.
**Caution:** it only bypasses the *policy* check — RSVP signalling and reachability still
apply, so a verbatim path can still fail with a *Signalled* error if the network can't build it.

### `auto-bw` block — [PER-TUNNEL] (headend)
```
auto-bw
 application <minutes>            ! resize interval (5 min min; default 1440 = 24h)
 bw-limit min <kbps> max <kbps>   ! clamp the auto-sized reservation
 adjustment-threshold <percent>   ! only resize if measured rate differs by > this
```
**Job:** the tunnel measures its own traffic and resizes `signalled-bandwidth` to match,
within min/max, make-before-break. **If missing:** the reservation stays at the static
value you guessed. Use a long `application` in production; short only for lab observation.

### `priority <setup> <hold>` — [PER-TUNNEL]
**Job:** DS-TE preemption priority, `0` (best) to `7` (worst), default `7 7`.
- **setup** = how aggressively this tunnel preempts others when establishing
- **hold** = how strongly it resists being preempted once up
**Rule:** a new tunnel preempts an existing one when `new setup < existing hold`. Give
critical tunnels low numbers; pair best-effort tunnels with a `dynamic` fallback so being
preempted means reroute, not outage.

---

## One-page summary

| Parameter | Scope | Key point |
|---|---|---|
| IS-IS TE extensions | domain-wide | floods bandwidth info; needed for CSPF |
| TE router-id | per-device | must match tunnel destination |
| RSVP bandwidth | per-interface | reservable pool; set to line rate or % of it |
| `mpls traffic-eng interface` | per-device | enables TE on that link |
| `reoptimize` | local | periodic re-computation; 300s default |
| `auto-tunnel backup` | per-PLR | tunnel-id pool; protection enabled **per-interface** |
| `ipv4 unnumbered mpls traffic-eng` | per-PLR | source for auto-tunnels; missing = backups down |
| `explicit-path` | per-tunnel | strict/loose hop list; wrong address = tunnel down |
| `path-option` (+`verbatim`) | per-tunnel | ordered fallback list; `verbatim` skips CSPF/affinity |
| `signalled-bandwidth` | per-tunnel | kbps reserved; too high = no path found |
| `autoroute announce` | per-tunnel | injects tunnel into routing table |
| `fast-reroute` | per-tunnel | enables FRR; needs bypass tunnels on PLRs |
| `affinity-map` | domain-wide | name → admin-group bit; must match everywhere |
| `attribute-names` | per-link | paints a link with a color |
| `affinity include/exclude` / `0x0 mask 0x0` | per-tunnel | steer by color; default is "uncolored only" |
| `auto-bw` | per-tunnel | resizes reservation from measured traffic |
| `priority <setup> <hold>` | per-tunnel | DS-TE preemption; 0 best, 7 worst |
