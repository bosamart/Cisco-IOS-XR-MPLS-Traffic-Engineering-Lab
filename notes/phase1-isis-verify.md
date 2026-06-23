# Phase 1 — IS-IS Baseline: Verification Log

**Date:** 2026-06-23
**Objective:** Confirm IS-IS L2 adjacencies are up on all five core links and the
link-state database is fully synchronized before introducing any MPLS.

**Result:** ✅ **PASS** — 5/5 core links adjacent, LSDB synchronized on all four routers.

---

## Commands run (per router)

```
show isis neighbors
show isis adjacency
show isis database
show isis interface brief      ! NOTE: 'show clns interface brief' is IOS, not IOS-XR — it errors on XR
```

---

## Expected adjacency map

| Router | Neighbors (interface) | Count |
|--------|------------------------|-------|
| R1 | R2 (Gi0/0/0/1), R3 (Gi0/0/0/3) | 2 |
| R2 | R1 (Gi0/0/0/0), R3 (Gi0/0/0/1), R4 (Gi0/0/0/3) | 3 |
| R3 | R1 (Gi0/0/0/2), R2 (Gi0/0/0/0), R4 (Gi0/0/0/4) | 3 |
| R4 | R2 (Gi0/0/0/2), R3 (Gi0/0/0/3) | 2 |

All observed adjacencies matched this map. ✔

---

## Captured output

### R1

```
RP/0/RP0/CPU0:R1#show isis neighbors
IS-IS CORE neighbors:
System Id      Interface        SNPA           State Holdtime Type IETF-NSF
R2             Gi0/0/0/1        *PtoP*         Up    27       L2   Capable
R3             Gi0/0/0/3        *PtoP*         Up    26       L2   Capable
Total neighbor count: 2

RP/0/RP0/CPU0:R1#show isis adjacency
IS-IS CORE Level-2 adjacencies:
System Id      Interface                SNPA           State Hold Changed  NSF IPv4 IPv6
R2             Gi0/0/0/1                *PtoP*         Up    27   00:03:07 Yes None None
R3             Gi0/0/0/3                *PtoP*         Up    26   00:05:08 Yes None None
Total adjacency count: 2

RP/0/RP0/CPU0:R1#show isis database
IS-IS CORE (Level-2) Link State Database
LSPID                 LSP Seq Num  LSP Checksum  LSP Holdtime/Rcvd  ATT/P/OL
R1.00-00            * 0x00000009   0xbab2        1012 /*            0/0/0
R2.00-00              0x0000000b   0x4572        1011 /1200         0/0/0
R3.00-00              0x00000007   0x0a96        920  /1200         0/0/0
R4.00-00              0x00000006   0x9c92        918  /1200         0/0/0
 Total Level-2 LSP count: 4     Local Level-2 LSP count: 1
```

### R2

```
RP/0/RP0/CPU0:R2#show isis neighbors
IS-IS CORE neighbors:
System Id      Interface        SNPA           State Holdtime Type IETF-NSF
R1             Gi0/0/0/0        *PtoP*         Up    29       L2   Capable
R4             Gi0/0/0/3        *PtoP*         Up    28       L2   Capable
R3             Gi0/0/0/1        *PtoP*         Up    26       L2   Capable
Total neighbor count: 3

RP/0/RP0/CPU0:R2#show isis adjacency
IS-IS CORE Level-2 adjacencies:
System Id      Interface                SNPA           State Hold Changed  NSF IPv4 IPv6
R1             Gi0/0/0/0                *PtoP*         Up    29   00:04:14 Yes None None
R4             Gi0/0/0/3                *PtoP*         Up    28   00:05:47 Yes None None
R3             Gi0/0/0/1                *PtoP*         Up    26   00:06:13 Yes None None
Total adjacency count: 3

RP/0/RP0/CPU0:R2#show isis database
IS-IS CORE (Level-2) Link State Database
LSPID                 LSP Seq Num  LSP Checksum  LSP Holdtime/Rcvd  ATT/P/OL
R1.00-00              0x00000009   0xbab2        947  /1200         0/0/0
R2.00-00            * 0x0000000b   0x4572        945  /*            0/0/0
R3.00-00              0x00000007   0x0a96        855  /1200         0/0/0
R4.00-00              0x00000006   0x9c92        852  /1199         0/0/0
 Total Level-2 LSP count: 4     Local Level-2 LSP count: 1
```

### R3

```
RP/0/RP0/CPU0:R3#show isis neighbors
IS-IS CORE neighbors:
System Id      Interface        SNPA           State Holdtime Type IETF-NSF
R1             Gi0/0/0/2        *PtoP*         Up    23       L2   Capable
R2             Gi0/0/0/0        *PtoP*         Up    23       L2   Capable
R4             Gi0/0/0/4        *PtoP*         Up    22       L2   Capable
Total neighbor count: 3

RP/0/RP0/CPU0:R3#show isis adjacency
IS-IS CORE Level-2 adjacencies:
System Id      Interface                SNPA           State Hold Changed  NSF IPv4 IPv6
R1             Gi0/0/0/2                *PtoP*         Up    23   00:06:32 Yes None None
R2             Gi0/0/0/0                *PtoP*         Up    23   00:06:32 Yes None None
R4             Gi0/0/0/4                *PtoP*         Up    22   00:06:06 Yes None None
Total adjacency count: 3

RP/0/RP0/CPU0:R3#show isis database
IS-IS CORE (Level-2) Link State Database
LSPID                 LSP Seq Num  LSP Checksum  LSP Holdtime/Rcvd  ATT/P/OL
R1.00-00              0x00000009   0xbab2        928  /1200         0/0/0
R2.00-00              0x0000000b   0x4572        926  /1200         0/0/0
R3.00-00            * 0x00000007   0x0a96        836  /*            0/0/0
R4.00-00              0x00000006   0x9c92        833  /1200         0/0/0
 Total Level-2 LSP count: 4     Local Level-2 LSP count: 1
```

### R4

```
RP/0/RP0/CPU0:R4#show isis neighbors
IS-IS CORE neighbors:
System Id      Interface        SNPA           State Holdtime Type IETF-NSF
R2             Gi0/0/0/2        *PtoP*         Up    26       L2   Capable
R3             Gi0/0/0/3        *PtoP*         Up    24       L2   Capable
Total neighbor count: 2

RP/0/RP0/CPU0:R4#show isis adjacency
IS-IS CORE Level-2 adjacencies:
System Id      Interface                SNPA           State Hold Changed  NSF IPv4 IPv6
R2             Gi0/0/0/2                *PtoP*         Up    26   00:06:27 Yes None None
R3             Gi0/0/0/3                *PtoP*         Up    24   00:06:27 Yes None None
Total adjacency count: 2

RP/0/RP0/CPU0:R4#show isis database
IS-IS CORE (Level-2) Link State Database
LSPID                 LSP Seq Num  LSP Checksum  LSP Holdtime/Rcvd  ATT/P/OL
R1.00-00              0x00000009   0xbab2        907  /1200         0/0/0
R2.00-00              0x0000000b   0x4572        906  /1200         0/0/0
R3.00-00              0x00000007   0x0a96        815  /1200         0/0/0
R4.00-00            * 0x00000006   0x9c92        812  /*            0/0/0
 Total Level-2 LSP count: 4     Local Level-2 LSP count: 1
```

---

## Analysis

- **Adjacencies:** All five core links formed L2 point-to-point adjacencies in
  state **Up** (R1:2, R2:3, R3:3, R4:2 = 10 endpoints = 5 bidirectional links).
  Matches the topology exactly — no missing or extra neighbors.
- **LSDB synchronization:** Every router holds the same 4 LSPs (R1–R4) with
  **identical sequence numbers and checksums** (R1 `0x...bab2`, R2 `0x...4572`,
  R3 `0x...0a96`, R4 `0x...9c92`). The database is fully flooded and converged.
- **Health flags:** `ATT/P/OL = 0/0/0` on every LSP — no attached bit, no
  overload bit. Clean L2-only domain.
- **Command note:** `show clns interface brief` returned *"Invalid input"* on all
  routers — that is the IOS form. On IOS-XR use `show isis interface brief`.

**Conclusion:** IS-IS baseline is solid. Safe to proceed to Phase 2 (LDP).

---

## Still to capture for a complete Phase 1 record

- `show route isis` on R1 — confirm remote loopbacks learned, ECMP to 4.4.4.4/32
- `ping 4.4.4.4 source 1.1.1.1` — end-to-end IGP reachability across the diamond
