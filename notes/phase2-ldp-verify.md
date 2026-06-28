# Phase 2 — LDP Label Distribution: Verification Log

**Date:** 2026-06-23
**Objective:** Confirm LDP forms a session to every IGP neighbor, allocates a label
for every prefix, and that the data plane label-switches end to end with PHP.

**Result:** ✅ **PASS** — LDP sessions match the IS-IS neighbor map (R1:2, R2:3, R3:3,
R4:2), labels allocated for all prefixes, traceroute shows an MPLS label in transit
popped before the tail (PHP).

---

## Commands run

```
show mpls ldp neighbor brief
show mpls ldp discovery
show mpls interfaces
show mpls ldp bindings            ! R1 (representative)
show mpls forwarding              ! R1 (representative)
traceroute 4.4.4.4 source 1.1.1.1 ! R1 — data-plane proof
```

---

## LDP sessions (all routers)

Each router formed an LDP session to exactly its IS-IS neighbors — same counts as Phase 1.

| Router | LDP peers | Count |
|--------|-----------|-------|
| R1 | 2.2.2.2, 3.3.3.3 | 2 |
| R2 | 1.1.1.1, 3.3.3.3, 4.4.4.4 | 3 |
| R3 | 1.1.1.1, 2.2.2.2, 4.4.4.4 | 3 |
| R4 | 2.2.2.2, 3.3.3.3 | 2 |

```
RP/0/RP0/CPU0:R1#show mpls ldp neighbor brief
Peer               GR  NSR  Up Time     Discovery   Addresses   Labels
3.3.3.3:0          N   N    00:13:47    1     0     4     0     9      0
2.2.2.2:0          N   N    00:11:53    1     0     4     0     9      0

RP/0/RP0/CPU0:R2#show mpls ldp neighbor brief
3.3.3.3:0          N   N    00:15:02    1     0     4     0     9      0
4.4.4.4:0          N   N    00:14:36    1     0     3     0     9      0
1.1.1.1:0          N   N    00:13:08    1     0     3     0     9      0

RP/0/RP0/CPU0:R3#show mpls ldp neighbor brief
2.2.2.2:0          N   N    00:15:24    1     0     4     0     9      0
1.1.1.1:0          N   N    00:15:24    1     0     3     0     9      0
4.4.4.4:0          N   N    00:14:58    1     0     3     0     9      0

RP/0/RP0/CPU0:R4#show mpls ldp neighbor brief
2.2.2.2:0          N   N    00:15:18    1     0     4     0     9      0
3.3.3.3:0          N   N    00:15:18    1     0     4     0     9      0
```

`show mpls ldp discovery` confirmed `xmit/recv` on every core interface with a
15s hold time; `show mpls interfaces` showed `LDP=Yes` on all core links (the
`Tunnel=Yes` column is MPLS-TE already enabled on the interface — see note below).

---

## Label bindings (R1)

```
RP/0/RP0/CPU0:R1#show mpls ldp bindings
1.1.1.1/32   Local: ImpNull         (own loopback -> implicit-null / PHP)
2.2.2.2/32   Local: 24001   Remote: 2.2.2.2=ImpNull, 3.3.3.3=24001
3.3.3.3/32   Local: 24000   Remote: 2.2.2.2=24000,  3.3.3.3=ImpNull
4.4.4.4/32   Local: 24005   Remote: 2.2.2.2=24005,  3.3.3.3=24004
10.12.0.0/30 Local: ImpNull Remote: 2.2.2.2=ImpNull, 3.3.3.3=24002
10.13.0.0/30 Local: ImpNull Remote: 2.2.2.2=24003,  3.3.3.3=ImpNull
10.23.0.0/30 Local: 24004   Remote: 2.2.2.2=ImpNull, 3.3.3.3=ImpNull
10.24.0.0/30 Local: 24003   Remote: 2.2.2.2=ImpNull, 3.3.3.3=24003
10.34.0.0/30 Local: 24002   Remote: 2.2.2.2=24002,  3.3.3.3=ImpNull
```

A local + remote label exists for every IGP prefix. `ImpNull` (implicit-null)
is advertised for directly-connected prefixes — this is what drives PHP.

---

## MPLS forwarding (R1)

```
RP/0/RP0/CPU0:R1#show mpls forwarding
Local  Outgoing    Prefix          Outgoing   Next Hop      Bytes
Label  Label       or ID           Interface                Switched
------ ----------- --------------- ---------- ------------- ----------
24000  Pop         3.3.3.3/32      Gi0/0/0/3  10.13.0.2     1686
24001  Pop         2.2.2.2/32      Gi0/0/0/1  10.12.0.2     1372
24002  Pop         10.34.0.0/30    Gi0/0/0/3  10.13.0.2     0
24003  Pop         10.24.0.0/30    Gi0/0/0/1  10.12.0.2     0
24004  Pop         10.23.0.0/30    Gi0/0/0/3  10.13.0.2     0
       Pop         10.23.0.0/30    Gi0/0/0/1  10.12.0.2     0
24005  24004       4.4.4.4/32      Gi0/0/0/3  10.13.0.2     3388
       24005       4.4.4.4/32      Gi0/0/0/1  10.12.0.2     0
24006  Aggregate   CUST-A: Per-VRF Aggr                     0
24007  Unlabelled  11.11.11.11/32  Gi0/0/0/0  192.168.11.2  0
```

- **Pop** entries = penultimate-hop popping for directly-connected prefixes.
- **4.4.4.4/32** is two hops away, so R1 **swaps** to the downstream label
  (24004 via R3, 24005 via R2) — and it's **ECMP** across both paths.
- 24006/24007 are L3VPN labels (CUST-A aggregate + CE1 prefix) — see note below.

---

## Data-plane proof: traceroute (R1)

```
RP/0/RP0/CPU0:R1#traceroute 4.4.4.4 source 1.1.1.1
 1  10.12.0.2 [MPLS: Label 24005 Exp 0] 36 msec   <-- label pushed in transit
    10.13.0.2 12 msec                              <-- ECMP second path
    10.12.0.2 4 msec
 2  10.34.0.2 12 msec                              <-- R4 reached, plain IP (PHP done)
    10.24.0.2 5 msec  *
```

This is the visible difference from Phase 1: hop 1 carries an **MPLS label (24005)**,
and by hop 2 (R4) the packet is **plain IP** — the label was popped at the
penultimate hop (PHP). ECMP is visible across both R2 and R3 paths.

---

## Analysis

- **Sessions:** LDP neighbor count matches IS-IS exactly on all four routers —
  no missing or stuck sessions. All `xmit/recv`, 15s hold. ✔
- **Label allocation:** Every IGP prefix has a local + remote binding;
  directly-connected prefixes advertise implicit-null (drives PHP). ✔
- **Forwarding:** MPLS FIB populated; pop for adjacent prefixes, swap (ECMP) for
  the far PE 4.4.4.4/32. ✔
- **Data plane:** traceroute proves labelled transit with PHP before the tail. ✔

**Note — cumulative config:** the device configs are cumulative (all base phases), so
R1's forwarding table already shows the L3VPN labels (`24006` CUST-A aggregate,
`24007` 11.11.11.11/32). In a strict phase-by-phase build these would not appear
until Phase 6. Likewise `show mpls interfaces` shows `Tunnel=Yes` because MPLS-TE
is already enabled on the interfaces (Phase 3). Harmless for verifying LDP itself.

**Conclusion:** Plain-MPLS LDP forwarding is working. Safe to proceed to Phase 3 (RSVP-TE).
