# MPLS-TE for Beginners — Concepts + Guided Build

A plain-language companion to this lab. The [`README`](../README.md) gives the phased
build and the [`TE-Config-Guide`](TE-Config-Guide.md) is the command cheat-sheet. This
file explains the **why** behind each piece, so the configs stop feeling like magic.

Read it top to bottom once, then keep it open while you build.

---

## 1. The one idea behind everything: transport vs service

A service-provider network does two separate jobs:

- **Transport** — getting a packet from one edge router to another across the core.
- **Service** — what the customer actually buys (an L3VPN, an internet feed, an L2
  circuit).

The whole point of MPLS is that these two are **decoupled**. The core just needs to
move a labelled packet from ingress to egress; it doesn't care what's inside. That's
why in this lab the *same* L3VPN service (Phase 6) can ride on top of plain LDP, an
RSVP-TE tunnel, or Segment Routing — only the transport changes.

Keep this split in your head: Phases 1–5 and 7 are all **transport**. Phase 6 is the
**service** that rides on it.

---

## 2. Why "Traffic Engineering" exists

By default, IP routing (IS-IS here) sends every packet down the **shortest path**. In
our diamond, R1 reaches R4 two ways — via R2 or via R3 — and IS-IS uses both equally
(equal-cost). That's fine until you have a reason to *not* use the shortest path:

- One path is congested and you want to push some traffic the long way round.
- A link is expensive (paid transit) and you want to avoid it.
- A customer pays for a low-latency path and you must keep them off the slow links.

Plain IP routing can't express "send *this* traffic *that* way." **Traffic Engineering
lets the edge router choose the path** instead of blindly following the IGP. MPLS-TE
does it by building a **tunnel** (an LSP) along a path you control, then steering
traffic into it.

---

## 3. Vocabulary, in plain English

| Term | What it really means |
|------|----------------------|
| **Label** | A short number stuck on the front of a packet. Routers forward by label instead of IP — faster and lets you pin a path. |
| **LSP** (Label-Switched Path) | The end-to-end path a labelled packet follows. A "tunnel" is an LSP. |
| **LDP** | The simple protocol that hands out one label per IGP prefix. No path control — just "plain MPLS" along the shortest path. (Phase 2) |
| **RSVP-TE** | The protocol that *signals* a TE tunnel hop-by-hop and **reserves bandwidth** on each link. This is what makes the path "engineered." (Phase 3) |
| **Headend** | The router where a tunnel starts and where the path is decided (here, R1). |
| **CSPF** (Constrained SPF) | Shortest-path-first, but the headend runs it itself over the *TE topology* and **honors constraints** (bandwidth, link colors). Output = a path. (Phase 4) |
| **Autoroute** | The switch that says "actually use this tunnel" — it puts the tunnel into the routing table so traffic follows it. Without it the tunnel is up but idle. (Phase 4) |
| **FRR** (Fast Reroute) | A pre-built backup path that the router upstream of a failure switches to in <50 ms, before the headend even notices. (Phase 5) |
| **PLR** (Point of Local Repair) | The router that performs the FRR switch — the one just upstream of the broken link/node. |
| **Affinity / admin-group** | A "color" on a link. Tunnels can be told "only use GOLD links" — path selection by policy. (Phase 7) |
| **VRF** | A separate routing table on a PE for one customer, so customers stay isolated. (Phase 6) |
| **RD / RT** | RD makes a customer's routes globally unique; RT controls which VRFs import them. (Phase 6) |
| **PE / P / CE** | Provider Edge (talks to customers), Provider core (transit only), Customer Edge (the customer's router). |

---

## 4. The lab in one picture

See [`diagrams/topology.svg`](../diagrams/topology.svg). The shape that matters:

- **R1 and R4** are PEs (customer-facing). **R2 and R3** are P routers (core only).
- There are **two equal-cost paths** R1→R4 (via R2, via R3) plus a **cross-link**
  R2–R3. That redundancy is what makes TE worth doing — there's always another way.
- CE1 (behind R1) and CE2 (behind R4) are the customer; they only speak plain BGP and
  know nothing about MPLS.

---

## 5. Guided build — what you're doing in each phase, and why

Build **one phase at a time**. After each, run the verify commands and make sure it
works before moving on. Each phase here links to its verification log in
[`../notes/`](../notes/) where you can see real output.

### Phase 1 — IS-IS baseline *(get the map right first)*

**Why:** MPLS needs a working IGP underneath. Every router must be able to reach every
loopback before labels mean anything. The loopbacks become the "addresses" your tunnels
and BGP sessions point at.

**What success looks like:** `show isis neighbors` shows every directly-connected
neighbor **Up**, and `ping 4.4.4.4 source 1.1.1.1` from R1 works.

### Phase 2 — LDP *(plain MPLS, no path control)*

**Why:** Before you engineer paths, get basic label switching working. LDP gives every
prefix a label and floods it to neighbors, so the core forwards by label along the
shortest path. This is the "before" picture — no TE yet.

**What success looks like:** `traceroute 4.4.4.4` from R1 shows an **MPLS label** in
transit, popped at the second-to-last hop (PHP) so R4 gets plain IP.

### Phase 3 — RSVP-TE tunnel *(your first engineered path)*

**Why:** Now you force a path. You define an **explicit path** (a list of hops) and
RSVP signals a tunnel along it — deliberately the *scenic* route R1→R2→R3→R4 instead of
the shortest one. Each router on the path stores state for the tunnel; that's what
"stateful" means and it's the big contrast with Segment Routing.

**Two things beginners trip on (both happened in this lab):**
1. Every router in the path must **flood TE info into IS-IS** (`mpls traffic-eng
   level-2-only` under the IGP) or the headend can't compute/verify the path → tunnel
   stays down with `Path: not valid`.
2. The tunnel needs a **path-option**. No path-option = nothing to signal.

**What success looks like:** `show mpls traffic-eng tunnels brief` → `up/up`, and the
tunnel detail's **Record Route** lists R1→R2→R3→R4. (Don't trust a plain traceroute yet
— see Phase 4.)

### Phase 4 — CSPF + autoroute *(let the router pick, then actually use it)*

**Why:** Two upgrades. **CSPF** lets the headend compute a path itself (honoring
bandwidth) instead of you hard-coding hops. **Autoroute** is the crucial switch that
injects the tunnel into the routing table — *without it the tunnel is up but carries
nothing.*

**What success looks like:** `show route 4.4.4.4/32` now says **`via tunnel-te1`**, and
the traceroute finally collapses from two equal-cost paths to the single tunnel path.

### Phase 5 — FRR *(survive a failure in milliseconds)*

**Why:** A signaled tunnel still breaks if a link on its path dies — and waiting for the
headend to recompute is too slow for voice/video. FRR pre-builds a **bypass tunnel**
around the protected link/node; the router next to the failure switches to it instantly.

**Beginner reality check (all real gotchas from this lab):**
- Auto-built backup tunnels need a **source address**:
  `ipv4 unnumbered mpls traffic-eng Loopback0`. Without it, every bypass sits down with
  *"No IP source address is configured."*
- Protection is enabled **per interface** (on the link the tunnel actually uses) — not
  just globally.
- **Node** protection only works where there's a node *beyond* the next hop to bypass
  to. In our diamond that's R2 (it can jump straight to R4); R3 can't, because its next
  hop *is* the tail.

**What success looks like:** `show mpls traffic-eng fast-reroute database` shows the LSP
**Ready**; shut the protected link and it flips to **Active** with near-zero packet loss.

### Phase 6 — L3VPN over the tunnel *(the actual customer service)*

**Why:** Everything so far was transport. Now you sell something. A **VRF** keeps the
customer's routes separate; **MP-BGP (VPNv4)** carries them between PEs; the customer
prefix gets a **VPN label** that rides *inside* your TE tunnel. The headline proof is
that R1's forwarding entry for the remote customer resolves **via tunnel-te1**.

**What success looks like:** CE1 pings CE2 (`ping 22.22.22.22 source 11.11.11.11`), and
`show cef vrf CUST-A 22.22.22.22` on R1 shows the path resolving through the tunnel.

### Phase 7 — Affinity coloring *(steer by policy, not by hand)*

**Why:** Explicit paths (Phase 3) are brittle — you hard-code hops. **Affinity** is the
grown-up way: color your links (GOLD = north path, BRONZE = south path), then tell a
tunnel `affinity include GOLD`. CSPF only considers GOLD links and finds the north path
on its own. Change one line to `include BRONZE` and it re-routes to the south path — no
hops touched.

**What success looks like:** the tunnel's computed path moves from R1→R2→R4 (GOLD) to
R1→R3→R4 (BRONZE) when you flip the color, and the change happens **make-before-break**
(old path stays up until the new one is ready).

---

## 6. How to debug when a tunnel won't come up

This lab hit almost every common failure. A beginner's checklist, in order:

1. **Is the IGP healthy?** `show isis neighbors` — no adjacency, nothing else works.
2. **Is TE flooded everywhere on the path?**
   `show mpls traffic-eng link-management interface` should say **flooded** on every
   router the tunnel crosses. "Not flooded" = missing `mpls traffic-eng` under IS-IS.
3. **Does the tunnel have a valid path-option?** `show mpls traffic-eng tunnels N
   detail` — `Path: not valid` usually means missing path-option or an explicit hop the
   TE topology can't confirm.
4. **For FRR:** auto-tunnels down with *"No IP source address"* → add `ipv4 unnumbered
   mpls traffic-eng Loopback0`.
5. **For L3VPN:** session up but `0 prefixes`? Check the PE–CE link is actually cabled
   to the configured interface (XRv shows `Up/Up` even on an unconnected NIC — don't
   trust it; ping across the link).

> **IOS-XR habit to build now:** a bare `!` is a *comment*, not a submode exit. Use
> `exit` to step up one level or `root` to jump to global config. Several failures in
> this lab were just commands landing in the wrong submode.

---

## 7. Where to go next

- Run each phase and paste real output into the matching `notes/phaseN-*.md` log.
- Compare side-by-side with the **SR-MPLS lab** (same topology) to feel why the industry
  moved from RSVP-TE to Segment Routing — less core state, automatic TI-LFA, simpler FRR.
- Once comfortable, try `affinity exclude` (avoid a color) and add `signalled-bandwidth`
  so CSPF must satisfy color *and* capacity at once.
