# Phase 6 — L3VPN over MPLS-TE: Verification Log

**Date:** 2026-06-23
**Objective:** Carry VRF `CUST-A` (CE1↔CE2) across the core, proving the TE tunnel is
the transport for a real customer service.

**Result:** ✅ **PASS** — CE1↔CE2 ping 100%; R1's CEF for `22.22.22.22/32` resolves
recursively over **tunnel-te1** with the VPN label inside it.

---

## Config gap (a wiring/config mismatch, not a paste error)

The CE2↔R4 link was cabled in EVE-NG as `R4 Gi0/0/0/0 ↔ CE2 Gi0/0/0/1`, but the
config put the L3VPN IPs on `R4 Gi0/0/0/1` and `CE2 Gi0/0/0/0` — so the addressed
interfaces weren't actually connected, and the cabled ones had no config. (XRv9000
shows `Up/Up` even on an unconnected virtual NIC, which hid it.) eBGP could never form.

**Fix:** rewired the EVE link to match the documented design — `R4 Gi0/0/0/1 ↔
CE2 Gi0/0/0/0`. After that, ping and eBGP came up immediately.

> **Lesson:** if a PE–CE eBGP session is stuck and the interfaces show `Up/Up` but
> can't ping, check the EVE cabling against the config port-by-port. Don't trust
> XRv `Up/Up` as proof of L2.

---

## PE–CE eBGP established

```
RP/0/RP0/CPU0:R4#ping vrf CUST-A 192.168.44.2
!!!!!  Success rate is 100 percent (5/5)

RP/0/RP0/CPU0:R4#show bgp vrf CUST-A ipv4 unicast neighbor 192.168.44.2
 Remote AS 65002, local AS 100, external link
  BGP state = Established, up for 00:00:16
```

## Prefixes exchanged

```
! CE2 advertises its loopback:
RP/0/RP0/CPU0:CE2#show bgp ipv4 unicast neighbor 192.168.44.1 advertised-routes
Network            Next Hop        From     AS Path
22.22.22.22/32     192.168.44.2    Local    65002i

! R4 imports it into CUST-A and learns R1's side over VPNv4:
RP/0/RP0/CPU0:R4#show bgp vrf CUST-A ipv4 unicast
   Network            Next Hop        Metric LocPrf Weight Path
*>i11.11.11.11/32     1.1.1.1              0    100      0 65001 i    (from R1)
*> 22.22.22.22/32     192.168.44.2         0             0 65002 i    (from CE2)
*>i192.168.11.0/30    1.1.1.1              0    100      0 ?
*> 192.168.44.0/30    0.0.0.0              0         32768 ?

! R1 learns 22.22.22.22 over VPNv4:
RP/0/RP0/CPU0:R1#show route vrf CUST-A
B    11.11.11.11/32 [20/0]  via 192.168.11.2            (local CE1)
B    22.22.22.22/32 [200/0] via 4.4.4.4 (nexthop in vrf default)   (remote CE2)
C    192.168.11.0/30 is directly connected, Gi0/0/0/0
B    192.168.44.0/30 [200/0] via 4.4.4.4 (nexthop in vrf default)
```

## The key proof — VPN rides the TE tunnel

```
RP/0/RP0/CPU0:R1#show cef vrf CUST-A 22.22.22.22
22.22.22.22/32 ...
   via 4.4.4.4/32, recursive
    next hop 4.4.4.4/32 via 24002/0/21
     next hop 4.4.4.4/32 tt1          labels imposed {ImplNull 24007}
```

`tt1` = **tunnel-te1**. The VPN prefix resolves recursively to the BGP next-hop
(4.4.4.4 = R4), which forwards **over the TE tunnel**. The label stack is
`{ImplNull 24007}`: the inner **24007** is R4's VPNv4 label for CUST-A's route to
CE2; the transport is the TE tunnel (ImplNull at this resolution point). Customer
traffic physically rides the scenic RSVP-TE LSP R1→R2→R3→R4.

## End-to-end service

```
RP/0/RP0/CPU0:CE1#ping 22.22.22.22 source 11.11.11.11
!!!!!  Success rate is 100 percent (5/5), round-trip min/avg/max = 6/13/40 ms
```

---

## Analysis

- **PE–CE eBGP up** on both ends (R1↔CE1 AS65001, R4↔CE2 AS65002). ✔
- **VPNv4 distribution working** — CE prefixes cross the core in VRF CUST-A via
  MP-BGP between R1 and R4. ✔
- **Transport is the TE tunnel** — R1's CEF for the remote CE prefix resolves over
  `tunnel-te1`, VPN label nested inside. This is the headline: the service config
  (VRF/RD/RT/PE-CE BGP) is identical to the SR-MPLS lab; only the transport changed
  (RSVP-TE tunnel instead of SR/LDP). Services are decoupled from transport. ✔
- **End-to-end ping 100%** CE1↔CE2. ✔

**Conclusion:** L3VPN over MPLS-TE works end to end. **All six phases verified.**
