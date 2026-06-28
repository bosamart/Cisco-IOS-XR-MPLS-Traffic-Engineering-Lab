# MPLS-TE Lab Notes

## Lab Goals
- Understand classic MPLS Traffic Engineering with RSVP-TE
- Compare RSVP-TE with SR-TE from the SR-MPLS lab
- Build foundation for understanding why operators moved to Segment Routing

## Topics Covered
- [x] Phase 1 — IS-IS baseline
- [x] Phase 2 — LDP label distribution
- [x] Phase 3 — RSVP-TE basic tunnel with explicit path
- [x] Phase 4 — CSPF path computation + autoroute
- [x] Phase 5 — FRR link and node protection
- [x] Phase 6 — L3VPN over MPLS-TE
- [x] Phase 7 — Affinity / admin-group coloring
- [x] Phase 8 — Auto-bandwidth
- [x] Phase 9 — DS-TE priority + preemption

## Lab Topology
> Same diamond as SR-MPLS lab: R1(PE) – R2(P) – R4(PE), R1 – R3(P) – R4, R2–R3 cross-link
> CE1 on R1, CE2 on R4

## GitHub Repo
https://github.com/bosamart/xrv9000-mpls-te-lab.git

## Related Lab
SR-MPLS lab: ../SR-MPLS/ — same topology, same services, SR transport instead of RSVP-TE

## Session Log
| Date | Phase | Topic | Outcome |
|------|-------|-------|---------|
| 2026-06-24 | Phase 1 | IS-IS baseline — all adjacencies up, loopbacks reachable, ECMP to 4.4.4.4 | ✅ Pass |
| 2026-06-24 | Phase 2 | LDP sessions up, labels distributed, traceroute shows label + PHP | ✅ Pass |
| 2026-06-24 | Phase 3 | RSVP-TE tunnel up on scenic path R1→R2→R3→R4 (RRO confirms); fixed TE flooding on R1/R2 + missing path-option | ✅ Pass |
| 2026-06-24 | Phase 4 | autoroute injects tunnel into RIB; traceroute collapses ECMP → single scenic path; fixed missing `autoroute announce` | ✅ Pass |
| 2026-06-24 | Phase 5 | FRR Ready→Active on R2 cross-link fail; LSP reroutes to bypass tt100. Fixes: README test i/f wrong, node-prot only at R2, missing `ipv4 unnumbered mpls traffic-eng Lo0` | ✅ Pass |
| 2026-06-24 | Phase 6 | CE1↔CE2 L3VPN ping 100%; R1 CEF for 22.22.22.22 resolves via tunnel-te1 (VPN label 24007 in tunnel). Fixed CE2-R4 EVE wiring mismatch | ✅ Pass |
| 2026-06-25 | Phase 7 | Affinity coloring: tunnel-te2 include GOLD → R1→R2→R4, include BRONZE → R1→R3→R4 (CSPF, no explicit path). Saw make-before-break reopt | ✅ Pass |
| 2026-06-25 | Phase 7b | explicit+affinity: color drops a reachable path; `verbatim` bypasses policy not signalling; **default affinity 0x0/0xffff broke tunnel-te1** → fixed with `affinity 0x0 mask 0x0`. Root cause of cross-link: R2 missing `mpls traffic-eng interface` | ✅ Pass |
| 2026-06-25 | Phase 8 | Auto-bandwidth on tunnel-te1: resized reservation 100M → 30M floor from measured traffic, make-before-break (LSP 5→6) | ✅ Pass |
| 2026-06-25 | Phase 9 | DS-TE preemption: te6 (setup 0) bumped te5 (hold 7) off R1→R3; 600M moved priority 7→0, te5 down with admission/bw PathErr | ✅ Pass |

> Detailed per-phase output: [`phase1-isis-verify.md`](phase1-isis-verify.md), [`phase2-ldp-verify.md`](phase2-ldp-verify.md), [`phase3-rsvp-te-verify.md`](phase3-rsvp-te-verify.md), [`phase4-cspf-autoroute-verify.md`](phase4-cspf-autoroute-verify.md), [`phase5-frr-verify.md`](phase5-frr-verify.md), [`phase6-l3vpn-verify.md`](phase6-l3vpn-verify.md), [`phase7-affinity-verify.md`](phase7-affinity-verify.md), [`phase7b-explicit-affinity-verify.md`](phase7b-explicit-affinity-verify.md), [`phase8-autobw-verify.md`](phase8-autobw-verify.md), [`phase9-preemption-verify.md`](phase9-preemption-verify.md)

**All nine phases verified end to end. ✅**

## Lessons Learned
- `show clns interface brief` is IOS, not IOS-XR — use `show isis interface brief` on XR.
- Configs are cumulative (all base phases), so Phase 2 forwarding already shows L3VPN
  labels (CUST-A aggregate, CE1 prefix) and `Tunnel=Yes` on interfaces. Expected.
- ImpNull (implicit-null) on directly-connected prefixes is what drives PHP.
- Phase 3: a TE tunnel needs BOTH IGP TE flooding on every router in the path AND a
  valid path-option on the headend. `Path: not valid` = check both. RRO in
  `show mpls te tunnels 1 detail` is the real proof of the path, not `traceroute`.
- Without `autoroute announce` (Phase 4) the tunnel is up but unused — plain
  traceroute to the destination still rides IGP/LDP.
