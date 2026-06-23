# Phase 4 — CSPF + Autoroute: Verification Log

**Date:** 2026-06-23
**Objective:** Inject `tunnel-te1` into the routing table with `autoroute announce`
so traffic to destinations behind R4 actually follows the TE tunnel.

**Result:** ✅ **PASS** — after adding the missing `autoroute announce`, `4.4.4.4/32`
resolves via `tunnel-te1` and the traceroute collapses from IGP ECMP to the single
scenic path R1→R2→R3→R4.

---

## Config gap (same partial-paste pattern)

First check showed `AutoRoute: disabled`, and CEF still resolved `4.4.4.4/32` over
plain IGP/LDP ECMP (via R2 *and* R3) — the tunnel was up but unused. The
`autoroute announce` line was missing from the running tunnel config (third missing
line from the same paste, after IS-IS TE flooding and the path-option). **Fix:**

```
interface tunnel-te1
 autoroute announce
commit
```

---

## Before vs after (R1)

**Before** — `4.4.4.4/32` via two physical next-hops (IGP/LDP ECMP):
```
via 10.13.0.2 ... labels imposed {24004}    (R3)
via 10.12.0.2 ... labels imposed {24005}    (R2)
```

**After** — `4.4.4.4/32` via the tunnel:
```
RP/0/RP0/CPU0:R1#show mpls traffic-eng tunnels 1 detail | include AutoRoute
    AutoRoute:  enabled  LockDown: disabled   Policy class: not set

RP/0/RP0/CPU0:R1#show route 4.4.4.4/32
Routing entry for 4.4.4.4/32
  Known via "isis CORE", distance 115, metric 20, type level-2
  Routing Descriptor Blocks
    4.4.4.4, from 4.4.4.4, via tunnel-te1       <-- single next-hop = the tunnel
      Route metric is 20

RP/0/RP0/CPU0:R1#show cef 4.4.4.4/32
   via 4.4.4.4/32, tunnel-te1, 3 dependencies, weight 0, class 0
    next hop 4.4.4.4/32
    local adjacency
     local label 24005      labels imposed {ImplNull}
    Load distribution: 0 (refcount 3)
    Hash  OK  Interface     Address
    0     Y   tunnel-te1    point2point
```

## Data-plane proof — traceroute now follows the tunnel

```
RP/0/RP0/CPU0:R1#traceroute 4.4.4.4 source 1.1.1.1
 1  10.12.0.2 [MPLS: Label 24004 Exp 0]   <-- R2
 2  10.23.0.2 [MPLS: Label 24005 Exp 0]   <-- R3 (cross-link)
 3  10.34.0.2                             <-- R4 (label popped, PHP)
```

Compare to Phase 1/2/3, where `traceroute 4.4.4.4` showed **two** paths (ECMP via
R2 and R3). Now it is a **single 3-hop scenic path** — the tunnel is in the
forwarding path.

---

## Analysis

- **Autoroute works.** `autoroute announce` makes the headend insert `tunnel-te1`
  as the next-hop for IGP destinations beyond the tail (R4), so the RIB/CEF point
  at the tunnel instead of the physical ECMP paths. ✔
- **Single scenic path in the data plane.** Traceroute confirms R1→R2→R3→R4 with
  label stack 24004 → 24005 → PHP — traffic now rides the deliberately longer
  (weight 30) TE path instead of the IGP shortest path (weight 20). ✔
- **Route metric preserved.** The tunnel route keeps metric 20 (autoroute announces
  the tunnel at the IGP cost of the tail), so other routing is unaffected. ✔

### Note on CSPF (dynamic path computation)

The tunnel is still set up on **path-option 1 (explicit SCENIC)**; `path-option 2
dynamic` is configured only as a fallback, so CSPF is not actively choosing the
path right now. The "autoroute" half of Phase 4 is fully demonstrated. To see CSPF
actively compute a constrained path, you'd make the dynamic option primary
(e.g. temporarily `path-option 1 dynamic`) or shut the explicit path — CSPF would
then run SPF over the TE topology (honoring the 100 Mbps constraint) and pick a
path itself. Optional to demo; not required for the autoroute objective.

**Conclusion:** Tunnel is injected into the routing table and carrying traffic on
the scenic path. Proceed to Phase 5 (FRR / fast reroute).
