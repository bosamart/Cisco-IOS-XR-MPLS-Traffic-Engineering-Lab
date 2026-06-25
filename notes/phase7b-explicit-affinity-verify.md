# Phase 7b — Explicit-path + Affinity Interaction (and `verbatim`): Verification Log

**Date:** 2026-06-25
**Objective:** see what happens when a tunnel is pinned to an explicit path *and*
constrained by color, learn what `verbatim` does, and discover the default-affinity trap.

**Result:** ✅ Three distinct lessons proven on a demo `tunnel-te3` — plus an unplanned
(and very real) one: **link coloring broke the production `tunnel-te1`** via default
affinity, and we fixed it.

---

## Lesson 1 — color can drop a perfectly reachable path

`tunnel-te3`: explicit path `SCENIC-R1-TO-R4` (R1→R2→R3→R4) + `affinity include GOLD`.
The scenic path crosses the uncolored cross-link and a BRONZE link — neither is GOLD.

```
Admin: up  Oper: down   Path: not valid
path option 1, type explicit SCENIC-R1-TO-R4
Last PCALC Error: No path to destination, 4.4.4.4 (unknown)
Include affinity name : GOLD(0)
```

`PCALC` = the headend's CSPF. It checked the explicit hops against the GOLD constraint
and **rejected the path before sending anything**. The path is fully reachable; only the
*color* drops it. (This is the fail-closed behaviour operators want for maintenance
drains: tag a link, `affinity exclude DRAIN`, and even pinned tunnels step off it.)

## Lesson 2 — `verbatim` bypasses policy, not signalling

Add `verbatim` to the same explicit path:

```
path option 1, (verbatim) type explicit SCENIC-R1-TO-R4
Last Signalled Error: PathErr(24,5) no route to dest (5) at 10.12.0.2
```

The error type **changed** — from a `PCALC Error` (Lesson 1) to a **`Signalled Error`**.
That proves `verbatim` did its job: it skipped the CSPF/affinity check and let RSVP
**actually try to signal** the hops. RSVP then hit a real lower-layer problem on the
cross-link path. So:

> `verbatim` removes the *policy* objection (color/CSPF). It cannot manufacture a path
> the network can't build. Lesson 1 = "policy says no." Lesson 2 = "policy bypassed —
> now really try — and signalling says no."

For contrast, pinning an **all-GOLD** explicit path (`NORTH-R1-TO-R4` = 10.12.0.2,
10.24.0.2) with the same `include GOLD` came **up** immediately (weight 20, R1→R2→R4) —
because now every hop satisfies the color, so CSPF passes and it signals cleanly.

## Lesson 3 (the big one) — default affinity is "avoid ALL colored links"

While testing, the **production `tunnel-te1` (which has NO affinity configured) went
down**:

```
Bandwidth: 100000 kbps   Affinity: 0x0/0xffff       <-- the DEFAULT, not blank
path option 1 ... No path to destination, 4.4.4.4 (affinity)
path option 2 ... No path to destination, 4.4.4.4 (affinity)
```

Every tunnel carries a **default affinity of `0x0/0xffff`**, which means *"every link I
use must have all admin-group bits = 0"* — i.e. **uncolored links only**. The moment
Phase 7 colored the core links GOLD (bit 0) / BRONZE (bit 1), `tunnel-te1` could no
longer find an all-uncolored R1→R4 path and failed with `(affinity)` — even though it
never mentions affinity.

> **Real-world incident shape:** "I added link coloring and unrelated tunnels dropped."
> Anything relying on the default affinity will refuse the newly-colored links.

**Fix** — tell the tunnel to ignore color bits (mask = 0):

```
interface tunnel-te1
 affinity 0x0 mask 0x0        ! mask 0x0 = check no bits = ride any link, colored or not
commit
```

**Verify — back up, service restored:**
```
RP/0/RP0/CPU0:R1#show mpls traffic-eng tunnels 1 brief
   tunnel-te1   4.4.4.4   up   up

RP/0/RP0/CPU0:CE1#ping 22.22.22.22 source 11.11.11.11
!!!!!  Success rate is 100 percent (5/5)
```

(This fix is now in `configs/R1.txt`. Without it, the coloring in the final config
leaves `tunnel-te1` down — so it's a required part of the lab, not optional.)

---

## Takeaways

- An explicit-path option is **CSPF-verified by default**, including affinity — a
  wrong-colored hop drops it. `verbatim` turns that verification off (signals as-is).
- `verbatim` only defeats the *policy* check; signalling/reachability still applies.
- **Default tunnel affinity `0x0/0xffff` = "uncolored links only."** Introducing link
  colors can silently break tunnels that don't set their own affinity. Give such tunnels
  `affinity 0x0 mask 0x0` (ignore colors) or an explicit `include` that matches.

## Cleanup (tunnel-te3 is a teaching tunnel)

```
no interface tunnel-te3
no explicit-path name NORTH-R1-TO-R4
```

## Lesson 4 — IS-IS, RSVP, and `mpls traffic-eng interface` are THREE separate layers

After the affinity fix, the scenic path (and `tunnel-te1`'s preferred path-option 1) still
wouldn't come up — failing at **PCALC** with `No path to destination (unknown)`: CSPF
couldn't even compute the path, never mind signal it. Chasing it down:

- `tunnel-te4` pinned to SCENIC (colors ignored, no fallback) failed at **PCALC**, not
  signalling — so the path can't be *computed* = a TE-topology problem.
- R1's TE topology had only **one direction** of the R2↔R3 cross-link (the half advertised
  by R3): `show mpls traffic-eng topology brief | include 10.23` showed just `10.23.0.2`.
- **R2 link-management showed `Links Count: 2`** (should be 3). R2 was **not** advertising
  the cross-link `Gi0/0/0/1` into TE — even though IS-IS was adjacent and RSVP was enabled.

**Root cause:** R2's running config had drifted (post-restart) and lost
`mpls traffic-eng interface GigabitEthernet0/0/0/1`. A link needs **three independent
things** to be usable for TE, all on the same wire:

| Layer | Command | If missing... |
|-------|---------|---------------|
| IS-IS adjacency | `router isis / interface` | IGP can't route over it |
| RSVP | `rsvp interface` | can't signal an LSP across it (Signalled error) |
| TE advertisement | `mpls traffic-eng interface` | link invisible to CSPF (PCALC "no path") |

R2↔R3 had the first two but not the third. **Fix:** re-add
`mpls traffic-eng interface GigabitEthernet0/0/0/1` on R2 → link count went to 3, cross-link
flooded, `tunnel-te4` came up on SCENIC (RRO `10.12.0.2 → 10.23.0.2 → 10.34.0.2`), and
`tunnel-te1` reoptimized back to its preferred scenic path-option.

> **Senior-level takeaway:** read the *error type*. `PCALC Error` = CSPF / TE topology
> (is the link advertised? affinity? bandwidth?). `Signalled Error` = RSVP along the path.
> They point you at different layers. The earlier `verbatim` "no route to dest" was a
> transient red herring; the persistent fault was this TE-advertisement gap.

(The repo `configs/R2.txt` already carries `mpls traffic-eng interface Gi0/0/0/1` — this was
live-router drift after the 2-day restart, not a file bug.)
