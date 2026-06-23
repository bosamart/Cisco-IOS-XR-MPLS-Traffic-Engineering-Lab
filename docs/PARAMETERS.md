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

### `auto-tunnel backup tunnel-id min 100 max 200` — [PER-DEVICE] (on PLRs)
**Job:** automatically creates RSVP bypass tunnels for every MPLS-TE-enabled interface,
using tunnel IDs from the specified range.
**If missing:** FRR will not work even if `fast-reroute` is on the protected tunnel —
there is no bypass tunnel to redirect to.
**Must match?** Enable on every P router (the PLRs). The tunnel-id range must not
overlap with manually configured tunnel-te interfaces.

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

---

## One-page summary

| Parameter | Scope | Key point |
|---|---|---|
| IS-IS TE extensions | domain-wide | floods bandwidth info; needed for CSPF |
| TE router-id | per-device | must match tunnel destination |
| RSVP bandwidth | per-interface | reservable pool; set to line rate or % of it |
| `mpls traffic-eng interface` | per-device | enables TE on that link |
| `reoptimize` | local | periodic re-computation; 300s default |
| `auto-tunnel backup` | per-PLR | auto-creates bypass tunnels; needed for FRR |
| `explicit-path` | per-tunnel | strict hop list; wrong address = tunnel down |
| `path-option` | per-tunnel | ordered fallback list; always add `dynamic` as last |
| `signalled-bandwidth` | per-tunnel | kbps reserved; too high = no path found |
| `autoroute announce` | per-tunnel | injects tunnel into routing table |
| `fast-reroute` | per-tunnel | enables FRR; needs bypass tunnels on PLRs |
