# RSVP-TE vs SR-TE — The Full Comparison

Both labs use the same topology, the same services, the same IGP. The only difference
is *how the TE path is encoded and signaled*. This document makes that difference
crystal clear.

---

## The core difference in one sentence

**RSVP-TE:** every router on the path holds state for every tunnel passing through it.
**SR-TE:** only the headend holds the policy. Transit routers are stateless.

---

## Side-by-side

| Feature | RSVP-TE (this lab) | SR-TE (SR-MPLS lab) |
|---|---|---|
| Path encoding | IP hop addresses (`explicit-path`) | MPLS label stack (`segment-list`) |
| Signaling protocol | RSVP PATH/RESV messages, hop by hop | None — headend pushes labels onto packet |
| State in core | Yes — every transit router stores tunnel state | No — transit routers see only labels |
| Bandwidth reservation | Yes — RSVP reserves BW at each hop | No native BW reservation (PCE/controller needed) |
| Path computation | CSPF on headend (reads flooded TE topology) | CSPF on headend or PCE (same TE topology) |
| Fast reroute | Pre-signaled bypass tunnels (RSVP FRR) | TI-LFA (automatic, no bypass tunnel config) |
| Bypass tunnel setup | Manual or `auto-tunnel backup` | Automatic (IS-IS computes repair path) |
| Scale in core | O(tunnels × hops) — grows with tunnel count | O(tunnels) at headend only |
| Config complexity | High — RSVP + TE interfaces + explicit-path + tunnel | Low — segment-list + SR-TE policy |
| Troubleshooting | `show rsvp session` on every hop | `show sr-te policy` on headend only |
| Hardware requirement | Any MPLS-capable router | SR-capable (prefix-SID in IGP) |

---

## State explosion — why RSVP-TE doesn't scale

Imagine a network with 100 PEs, each with 50 tunnels to other PEs = 5,000 tunnels.
Each tunnel traverses 3–5 P routers.

- Every P router holds RSVP state for every tunnel passing through it.
- Each RSVP state entry needs a refresh message every 30 seconds (default).
- A P router in the middle might have 2,000–4,000 RSVP sessions to maintain.

This is why large operators ran into RSVP scalability walls in the 2010s.

With SR-TE, those 5,000 policies live only on the 100 PE headends.
P routers just forward labels — they hold zero tunnel state.

---

## Path encoding: explicit-path vs segment-list

**RSVP-TE explicit-path** for the scenic R1→R2→R3→R4 path:
```
explicit-path name SCENIC-R1-TO-R4
 index 10 next-address strict ipv4 unicast 10.12.0.2   ! R2's IP toward R1
 index 20 next-address strict ipv4 unicast 10.23.0.2   ! R3's IP toward R2
 index 30 next-address strict ipv4 unicast 10.34.0.2   ! R4's IP toward R3
```
You're listing *interface addresses* — the exact next-hop on each link.
If you renumber a link, you must update the explicit-path.

**SR-TE segment-list** for the same path:
```
segment-list SCENIC-R1-TO-R4
 index 10 mpls label 16002   ! prefix-SID of R2
 index 20 mpls label 16003   ! prefix-SID of R3
 index 30 mpls label 16004   ! prefix-SID of R4
```
You're listing *router identities* (SIDs). If you renumber a link, nothing changes —
the SID follows the router's loopback, not the link address.

---

## Fast reroute comparison

**RSVP FRR:**
1. PLR detects failure (link-down event)
2. PLR looks up the bypass tunnel for that interface
3. Switches traffic to bypass (< 50ms)
4. Meanwhile, headend re-signals the primary tunnel around the failure
5. When new primary is up, traffic moves back

The bypass tunnel is itself an RSVP-TE tunnel — it was pre-signaled, and it holds
state on every router it passes through too.

**TI-LFA (SR-MPLS lab):**
1. IS-IS detects failure
2. IS-IS has pre-computed a loop-free repair path (expressed as a label stack)
3. Traffic immediately uses the repair — same < 50ms
4. IS-IS reconverges; repair is no longer needed

No bypass tunnel configuration. No extra RSVP state. The repair path is encoded in
the data plane as SR labels — no separate signaling needed.

---

## When RSVP-TE is still the right answer

Despite SR-TE's advantages, RSVP-TE is still in production at many operators:

1. **Bandwidth reservation** — RSVP-TE provides hard per-tunnel bandwidth admission
   control natively. SR-TE requires a controller (PCE) to do this.
2. **Legacy networks** — networks built before SR where every router doesn't support
   segment routing.
3. **Hardware** — older line cards that can forward MPLS but not SR extensions.
4. **Interop** — some multi-vendor environments where SR implementation differs.

The real-world answer for most operators today is: **RSVP-TE for existing tunnels,
SR-TE for new tunnels, migration over time.**

---

## The key takeaway

Both labs run the same L3VPN. The service is identical. The transport is different.

That is the whole point of the "services are decoupled from transport" idea.
If you understand why RSVP-TE works, why it has problems at scale, and what SR-TE
does differently — you understand the last 20 years of SP networking evolution.
