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

## Phase 5 — FRR (all P routers = PLRs, plus headend)

```
! On every P router (R2, R3, R4):
mpls traffic-eng
 auto-tunnel backup
  tunnel-id min 100 max 200
 !
!

! On headend tunnel (R1 tunnel-te1):
interface tunnel-te1
 fast-reroute
 fast-reroute protect bandwidth
 fast-reroute protect node
!
```

**Verify**
```
show mpls traffic-eng tunnels backup       ! bypass tunnels auto-created
show mpls traffic-eng fast-reroute database ! protected prefixes
show mpls traffic-eng tunnels protection   ! per-tunnel FRR status

! Prove it live:
ping 4.4.4.4 source 1.1.1.1 count 1000
! (on R2 mid-ping) interface GigabitEthernet0/0/0/0 -> shutdown
! expect near-zero loss
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
