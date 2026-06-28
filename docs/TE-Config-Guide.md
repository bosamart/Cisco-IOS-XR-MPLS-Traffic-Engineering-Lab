# MPLS-TE Configuration Quick-Reference — Cisco IOS XR

All commands needed for each phase in one place. Copy-paste friendly.

---

## Phase 2 — LDP (all routers)

```
mpls ldp
 router-id <loopback-ip>
 interface GigabitEthernet0/0/0/X   ! repeat per core interface
```

**Verify**
```
show mpls ldp neighbor               ! sessions up
show mpls ldp bindings               ! label per prefix
show mpls forwarding                 ! MPLS FIB
traceroute <remote-loopback> source <local-loopback>  ! labels in transit
```

---

## Phase 3 — RSVP-TE: enable on all routers

```
! Enable RSVP on each core interface
rsvp
 interface GigabitEthernet0/0/0/X
  bandwidth 1000000    ! kbps — set to line rate
 !
!

! Enable MPLS-TE on each core interface
mpls traffic-eng
 interface GigabitEthernet0/0/0/X
 !
!

! Tell IS-IS to flood TE topology
router isis CORE
 address-family ipv4 unicast
  mpls traffic-eng level-2-only
  mpls traffic-eng router-id Loopback0
 !
!
```

## Phase 3 — Create tunnel on headend (R1 only)

```
! Define the explicit path (hop-by-hop IP addresses of next-hop interfaces)
explicit-path name SCENIC-R1-TO-R4
 index 10 next-address strict ipv4 unicast 10.12.0.2
 index 20 next-address strict ipv4 unicast 10.23.0.2
 index 30 next-address strict ipv4 unicast 10.34.0.2
!

! Create the tunnel
interface tunnel-te1
 ipv4 unnumbered Loopback0
 destination 4.4.4.4
 path-option 1 explicit name SCENIC-R1-TO-R4
 path-option 2 dynamic    ! fallback: CSPF picks path
!
```

**Verify**
```
show mpls traffic-eng tunnels              ! tunnel-te1, state: up/up
show mpls traffic-eng tunnels detail       ! path taken, hop-by-hop
show rsvp session                          ! on R2/R3 — see tunnel state
traceroute tunnel-te1                      ! hops: R2 → R3 → R4
```

---

## Phase 4 — CSPF + autoroute (headend R1 only, modify tunnel-te1)

```
interface tunnel-te1
 autoroute announce             ! inject into routing table
 signalled-bandwidth 100000     ! kbps to reserve (100 Mbps)
 path-option 1 dynamic          ! let CSPF compute (replaces explicit if desired)
!
mpls traffic-eng
 reoptimize 300                 ! recompute path every 5 min
!
```

**Verify**
```
show mpls traffic-eng tunnels detail
show mpls traffic-eng topology            ! TE database with BW per link
show route 4.4.4.4/32 detail              ! via tunnel-te1
show cef 4.4.4.4/32 detail
show mpls traffic-eng link-management bandwidth-allocation
```

---

## Phase 5 — FRR (P routers = PLRs, plus headend)

```
! On every P router that auto-creates backups (R2, R3, R4) — REQUIRED source
! address for auto-tunnels, or they stay down ("No IP source address is configured"):
ipv4 unnumbered mpls traffic-eng Loopback0
!
mpls traffic-eng
 interface <ON-PATH-egress-interface>   ! e.g. R2 Gi0/0/0/1 (the cross-link)
  auto-tunnel backup                    ! per-interface enables protection on THIS link
 !
 auto-tunnel backup
  tunnel-id min 100 max 200             ! global = just the tunnel-id pool
 !
!

! On headend tunnel (R1 tunnel-te1):
interface tunnel-te1
 fast-reroute
 fast-reroute protect node
!
```

> **Node protection only works where there's a next-next hop to bypass to.** In the
> diamond that's **R2** (it has a direct link to the tail R4); R3 can only do link
> protection (its next hop *is* the tail).

**Verify**
```
show mpls traffic-eng tunnels backup        ! bypass tunnels auto-created, Oper: up
show mpls traffic-eng fast-reroute database ! protected LSP -> bypass, state Ready
show mpls traffic-eng tunnels protection    ! per-tunnel FRR status

! Prove it live — shut the ON-PATH link the LSP uses, NOT an off-path one:
ping 4.4.4.4 source 1.1.1.1 count 100000
! (on R2 mid-ping) interface GigabitEthernet0/0/0/1 -> shutdown   ! the cross-link
! FRR database flips Ready -> Active; expect near-zero loss
```

---

## Phase 6 — L3VPN over MPLS-TE (PE routers only)

Same as SR-MPLS Phase 5. No changes needed if autoroute is already configured.
The VPN label is transported inside the TE tunnel automatically.

```
! PE router (R1, R4):
vrf CUST-A
 address-family ipv4 unicast
  import route-target 100:1
  export route-target 100:1
!
route-policy PASS
  pass
end-policy
!
router bgp 100
 address-family vpnv4 unicast
 neighbor <remote-PE-loopback>
  remote-as 100
  update-source Loopback0
  address-family vpnv4 unicast
 vrf CUST-A
  rd 100:1
  address-family ipv4 unicast
   redistribute connected
  neighbor <CE-peer-ip>
   remote-as <CE-AS>
   address-family ipv4 unicast
    route-policy PASS in
    route-policy PASS out
```

**Verify**
```
show bgp vpnv4 unicast summary
show route vrf CUST-A
show cef vrf CUST-A 22.22.22.22 detail    ! resolves via tunnel-te1
! from CE1:
ping 22.22.22.22 source 11.11.11.11
```

---

## Phase 7 — Affinity / admin-group coloring

```
! Every router (affinity-map must be identical network-wide):
mpls traffic-eng
 affinity-map GOLD bit-position 0
 affinity-map BRONZE bit-position 1
 interface <north-link>
  attribute-names GOLD
 interface <south-link>
  attribute-names BRONZE
!
! Headend — steer a tunnel by color (CSPF, no explicit hops):
interface tunnel-te2
 ipv4 unnumbered Loopback0
 destination 4.4.4.4
 path-option 1 dynamic
 affinity include GOLD          ! -> north; 'include BRONZE' -> south
```

> **Watch out:** the DEFAULT tunnel affinity is `0x0/0xffff` = "uncolored links only."
> Once you color links, any tunnel WITHOUT its own affinity (e.g. tunnel-te1) can't find a
> path and goes down with `No path (affinity)`. Fix it with `affinity 0x0 mask 0x0`.

**Verify**
```
show mpls traffic-eng tunnels 2 detail     ! affinity constraint + chosen path
show mpls traffic-eng topology             ! Attribute Names per link
```

---

## Phase 8 — Auto-bandwidth (headend tunnel)

```
interface tunnel-te1
 signalled-bandwidth 100000        ! starting value
 auto-bw
  application 5                    ! resize every 5 min (lab; production = hours/24h)
  bw-limit min 30000 max 500000
  adjustment-threshold 5
```

**Verify**
```
show mpls traffic-eng tunnels auto-bw brief   ! Requested/Signalled/Highest BW + countdown
show mpls traffic-eng tunnels 1 detail        ! Bandwidth Requested resizes after each period
```

---

## Phase 9 — DS-TE priority + preemption (experiment)

```
! Two tunnels over the same path, only one fits the link's reservable BW.
interface tunnel-te5            ! low priority
 priority 7 7
 affinity 0x0 mask 0x0          ! ignore colors (south links are BRONZE)
 signalled-bandwidth 600000
 path-option 1 explicit name SOUTH-R1-TO-R4
!
interface tunnel-te6            ! high priority — preempts te5
 priority 0 0
 affinity 0x0 mask 0x0
 signalled-bandwidth 600000
 path-option 1 explicit name SOUTH-R1-TO-R4
```

Rule: a new tunnel preempts an existing one when `new setup < existing hold` (0 = best).

**Verify**
```
show mpls traffic-eng tunnels brief                          ! te6 up, te5 down
show mpls traffic-eng link-management bandwidth-allocation   ! BW moves priority 7 -> 0
show mpls traffic-eng tunnels 5 detail                       ! PathErr admission/bandwidth
```

---

## Useful show commands — all phases

```
! IGP
show isis neighbors
show isis topology

! LDP
show mpls ldp neighbor
show mpls ldp bindings
show mpls forwarding

! RSVP
show rsvp neighbor
show rsvp session
show rsvp interface

! MPLS-TE topology
show mpls traffic-eng topology
show mpls traffic-eng link-management interfaces
show mpls traffic-eng link-management bandwidth-allocation

! Tunnels
show mpls traffic-eng tunnels
show mpls traffic-eng tunnels detail
show mpls traffic-eng tunnels backup

! FRR
show mpls traffic-eng fast-reroute database
show mpls traffic-eng tunnels protection

! BGP/VPN
show bgp vpnv4 unicast summary
show route vrf CUST-A
show cef vrf CUST-A <prefix> detail
```
