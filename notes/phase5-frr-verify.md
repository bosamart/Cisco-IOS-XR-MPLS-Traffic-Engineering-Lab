# Phase 5 — FRR (Fast Reroute): Verification Log

**Date:** 2026-06-23
**Objective:** Protect `tunnel-te1` against an on-path link/node failure with a
pre-signaled bypass, so traffic reroutes locally (sub-50 ms) instead of waiting
for the headend to recompute.

**Result:** ✅ **PASS** — FRR database went **Ready → Active** when the protected
cross-link was failed; the LSP rerouted onto auto-bypass `tunnel-te100` (node
protection around R3). Took several fixes to get there — documented below.

---

## The README test interface was wrong

The README said to shut `R2 Gi0/0/0/3` — but that's the **off-path R2→R4 link**.
The LSP runs R1→R2→**R3**→R4 and leaves R2 via the **cross-link `Gi0/0/0/1`**.
Failing `Gi0/0/0/3` does nothing to the tunnel. **Correct test link: `R2 Gi0/0/0/1`.**

## Where node protection is even possible

The tunnel requests **Node** protection. A PLR node-protects by bypassing to the
*next-next* hop:
- **R3** can't node-protect — its next hop is R4, the tail; nothing lies beyond it.
  (R3's auto-backup stayed `Oper: down, Protected LSPs: 0`.)
- **R2** can — next hop R3, next-next hop R4, and R2 has a **direct link to R4**.
  So R2 bypasses R3 straight to R4. **R2 is the PLR for the demo.**

## Three fixes before FRR armed

1. **`auto-tunnel backup` not in running config on R2/R3** → "No protected interfaces."
   Re-applied the global block (`auto-tunnel backup / tunnel-id min 100 max 200`).
2. **Per-interface enablement on the wrong interface.** Protection only activates on
   an interface that has per-interface `auto-tunnel backup`. It had landed on the
   off-path `Gi0/0/0/3`; moved it to the on-path **`Gi0/0/0/1`** on R2.
3. **Auto-tunnels had no source address** → every bypass sat `Oper: down` with
   *"Reason for the tunnel being down: No IP source address is configured"* (`Src: 0.0.0.0`).
   **Fix (the key one):**
   ```
   ipv4 unnumbered mpls traffic-eng Loopback0
   ```
   This global command gives auto-created TE tunnels their source. Manual `tunnel-te1`
   worked without it (it has `ipv4 unnumbered Loopback0` on the interface); auto-tunnels
   need the global form. **This command is missing from the lab config files.**

---

## Armed state (before failure) — R2

```
RP/0/RP0/CPU0:R2#show mpls traffic-eng tunnels backup
tunnel-te100 (auto-tunnel backup)   Admin: up, Oper: up
 Src: 2.2.2.2, Dest: 4.4.4.4         <-- NNHOP node-protect bypass (R2->R4 around R3)
  Protected LSPs: 1   Inuse: 100000 kbps   Protected i/fs: Gi0/0/0/1
tunnel-te101 (auto-tunnel backup)   Admin: up, Oper: up
 Src: 2.2.2.2, Dest: 3.3.3.3         <-- NHOP link-protect bypass (backup choice)
  Protected LSPs: 0   Protected i/fs: Gi0/0/0/1

RP/0/RP0/CPU0:R2#show mpls traffic-eng fast-reroute database
LSP identifier        In-label  Out Intf:Label     FRR Intf:Label   Status
1.1.1.1 1 [3]         24004     Gi0/0/0/1:24005    tt100:Pop        Ready
```

## After failing R2 Gi0/0/0/1 — FRR Active

```
RP/0/RP0/CPU0:R2(config)#interface GigabitEthernet0/0/0/1
RP/0/RP0/CPU0:R2(config-if)# shutdown
RP/0/RP0/CPU0:R2(config-if)# commit

RP/0/RP0/CPU0:R2#show mpls traffic-eng fast-reroute database
LSP identifier        In-label  Out Intf:Label   FRR Intf:Label   Status
1.1.1.1 1 [3]         24004     tt100:Pop                         Active   <-- rerouted onto bypass

RP/0/RP0/CPU0:R2#show mpls traffic-eng tunnels backup
tunnel-te100 ... Protected LSPs: 1 (1 active)
```

Restored with `no shutdown` on `Gi0/0/0/1`; FRR returns to Ready and the primary LSP
re-optimizes back to the scenic path.

---

## Analysis

- **FRR engaged.** Status transitioned **Ready → Active** on link failure. The LSP's
  outgoing path switched from `Gi0/0/0/1:24005` to `tt100:Pop` — i.e. R2 locally
  repaired onto the pre-signaled bypass without headend involvement. ✔
- **Node protection delivered at R2.** Bypass `tunnel-te100` carries the LSP around
  the failed cross-link straight to R4 (the next-next hop), protecting against R3. ✔
- **This is the RSVP-TE vs SR comparison point.** RSVP FRR needs explicitly built
  (here, auto-) bypass tunnels and extra RSVP state at the PLR. TI-LFA in the SR lab
  computes the backup automatically from the IGP with no extra core state.

### Ping loss number (to finalize)

The control-plane proof (Ready→Active, 1 active protected LSP) is captured above.
To complete the data-plane headline, record the R1 ping success rate during the
failover: `ping 4.4.4.4 source 1.1.1.1 count 100000` (shut the link mid-ping).
Expected: near-zero loss (a packet or two). **[paste success rate here]**

**Conclusion:** RSVP-TE FRR works — local repair onto a node-protect bypass.
Proceed to Phase 6 (L3VPN over MPLS-TE).

---

## Config additions this phase needed (not yet in the lab files)

```
! global, on any router running auto-tunnel backup (R2 used here):
ipv4 unnumbered mpls traffic-eng Loopback0
!
mpls traffic-eng
 interface GigabitEthernet0/0/0/1     ! the ON-PATH interface to protect
  auto-tunnel backup
 !
 auto-tunnel backup
  tunnel-id min 100 max 200
```
