# Understanding MPLS Traffic Engineering — Concepts

A study companion. The README shows *how* to configure it; this explains *why* each
piece works, so you can troubleshoot and explain it in interviews.

---

## The one idea

**MPLS-TE is about making the network carry traffic the way you want, not just the
way the IGP shortest path says.**

Without TE, IS-IS routes all traffic on the shortest path. If that path is congested
but a longer path is empty, IS-IS doesn't care — it still uses the shortest path.
MPLS-TE solves this by letting you signal a *tunnel* along any path you choose,
reserving bandwidth along the way.

---

## The three building blocks

Every MPLS-TE deployment has exactly three components:

**1. Information distribution** — IS-IS (or OSPF) TE extensions flood the TE topology:
what bandwidth is available on each link, what the admin groups are, what the current
reservation is. Every router has a complete TE database.

**2. Path computation** — CSPF (Constrained Shortest Path First) on the headend.
The headend reads the TE database and runs a modified Dijkstra that finds the shortest
path *satisfying the constraints* (bandwidth ≥ X, avoid admin-group Y, etc.).

**3. Signaling** — RSVP-TE carries the PATH message hop-by-hop from headend to tail,
reserving resources at each hop. The tail sends a RESV message back, and labels are
distributed in the RESV. This is what makes RSVP-TE "stateful" — every router on the
path holds PATH and RESV state for every tunnel through it.

---

## Phase by phase — the why

**Phase 1 — IS-IS baseline.**
Same as the SR lab. You need reachability before labels.

**Phase 2 — LDP.**
LDP distributes labels for every IGP prefix — one label per destination, per neighbor.
This gives you basic MPLS forwarding (shortest path, no TE).
*Why do we set up LDP before TE?* Because LDP provides the fallback transport. If a
TE tunnel goes down and there's no path-option fallback, LDP-labeled traffic continues.

**Phase 3 — RSVP-TE: the tunnel is signaled.**
You define an `explicit-path` (a list of strict hop addresses) and configure a
`tunnel-te` interface on the headend with that path-option. RSVP carries a PATH
message along 10.12.0.2 → 10.23.0.2 → 10.34.0.2, each router installs state,
and the tail (R4) sends RESV back. The tunnel comes up. Traffic can now use it —
but only if you steer it in (Phase 4).

*The statefulness point:* run `show rsvp session` on R2 or R3. You will see an entry
for the tunnel. In the SR lab, R2 and R3 know nothing about the SR-TE policy on R1.
This is the core scalability difference.

**Phase 4 — CSPF + autoroute.**
`autoroute announce` puts tunnel-te1 into the routing table as a next-hop for R4's
loopback and anything behind it. Now BGP and normal IP traffic uses the tunnel.
`path-option 1 dynamic` lets CSPF choose the path — you no longer hard-code the hops.
`signalled-bandwidth` tells CSPF how much bandwidth to find, and RSVP reserves it.

*Why CSPF matters:* it prevents overloading. If link R1-R2 only has 200 Mbps free
and you ask for 300 Mbps, CSPF routes around it. IS-IS would blindly use it.

**Phase 5 — FRR.**
A TE tunnel can fail if a link or node fails. FRR pre-provisions a *bypass tunnel*
around every protected link/node. When the failure is detected (link-down event),
the PLR (point of local repair — the router upstream of the failure) immediately
switches traffic onto the bypass tunnel. No RSVP re-signaling needed — it's instant.

*auto-tunnel backup* tells RSVP to automatically create bypass tunnels for every
protected interface. This is simpler than manually defining each bypass.

*Compare to TI-LFA (SR lab):* TI-LFA is also pre-computed. The key difference is
that TI-LFA uses SR label stacks — no extra RSVP state anywhere. RSVP FRR adds
more RSVP state (bypass tunnels are themselves RSVP sessions).

**Phase 6 — L3VPN over TE.**
The VPN config is identical to the SR lab. When `autoroute announce` is on tunnel-te1
and 4.4.4.4 resolves via the tunnel, the VPN label is transported inside the tunnel's
label — you get the same two-label stack as always, but the outer label now belongs
to the TE tunnel, not an LDP path.

---

## Key terms

| Term | Meaning |
|---|---|
| **RSVP-TE** | Resource Reservation Protocol – Traffic Engineering. Signals tunnels and reserves bandwidth hop by hop |
| **LSP** | Label Switched Path — the tunnel from headend to tail |
| **Headend / Ingress** | Router that originates the tunnel (R1 in this lab) |
| **Tailend / Egress** | Router that terminates the tunnel (R4 in this lab) |
| **Transit / PLR** | Router in the middle (R2, R3). PLR = Point of Local Repair |
| **explicit-path** | A configured list of strict or loose hop addresses |
| **CSPF** | Constrained Shortest Path First — headend SPF with bandwidth/admin constraints |
| **TE database (TEDB)** | Each router's copy of the TE topology, flooded by IGP TE extensions |
| **autoroute announce** | Injects the tunnel into the routing table as a next-hop |
| **signalled-bandwidth** | The bandwidth RSVP will reserve along the path |
| **FRR / bypass tunnel** | Pre-signaled detour tunnel around a protected link or node |
| **PLR** | The router upstream of a failure that performs local repair |
| **NHOP bypass** | Bypass tunnel going to the next-next-hop — protects a link |
| **NNHOP bypass** | Bypass tunnel going to the next-next-next-hop — protects a node |
| **PHP** | Penultimate Hop Popping — second-to-last router removes label |
| **LDP** | Label Distribution Protocol — distributes labels for all IGP prefixes, shortest path only |

---

## Questions to be able to answer

**What is RSVP-TE and why does it create "state" in the core?**
RSVP-TE signals tunnels by sending PATH messages hop by hop. Each transit router
installs state — the incoming/outgoing label pair, the bandwidth reservation, the
timer for refreshing the state. The core "knows" about every tunnel, unlike SR-TE
where only the headend holds the policy.

**What does CSPF do that normal SPF doesn't?**
Normal SPF finds the shortest path by hop count or metric. CSPF runs SPF on the TE
topology database while pruning links that don't meet constraints (insufficient
bandwidth, wrong admin-group). The result is the shortest path that actually fits.

**What is autoroute and why do you need it?**
The TE tunnel exists as a signaled LSP, but IS-IS and BGP don't know about it by
default. `autoroute announce` tells the router to inject the tunnel as a next-hop for
the tailend's address (and destinations behind it). Without autoroute, traffic doesn't
use the tunnel.

**What is the difference between link protection and node protection in FRR?**
Link protection (NHOP bypass): the backup path goes around the failed link, still
passing through the downstream node. If the downstream node also fails, this doesn't
help.
Node protection (NNHOP bypass): the backup path goes around both the failed link and
the downstream node. Requires the bypass tunnel to reach further into the network.
Node protection is stronger but needs more bypass tunnels.

**Why did the industry move from RSVP-TE to SR-TE?**
State explosion: in a large network with thousands of tunnels, every transit router
holds state for every tunnel passing through it. SR-TE pushes state to the headend
only — the core is stateless. SR-TE is also simpler to provision (no explicit-path
IP addresses, just SID labels), integrates naturally with controllers (PCE/SDN), and
TI-LFA is automatic without extra bypass tunnel configuration.
