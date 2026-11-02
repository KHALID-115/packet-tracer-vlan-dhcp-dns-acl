# Segmented Small-Office Network (Cisco Packet Tracer)

A small-company network built in Cisco Packet Tracer with VLAN segmentation, inter-VLAN routing, centralized DHCP and DNS, and an ACL that isolates the guest network from internal resources.

## Objectives

- Segment the network into Management, Employees, Guests and Servers VLANs.
- Route between VLANs using Router-on-a-Stick.
- Serve IP addresses to all VLANs from one DHCP server using DHCP relay.
- Resolve internal names with a DNS server.
- Allow guests only DHCP and DNS, and block access to all internal networks.

## Topology

![Topology](screenshots/topology.png)

| Device | Model | Role |
|---|---|---|
| R1 | Cisco 2911 | Router-on-a-Stick, DHCP relay, ACL |
| SW1 | Cisco 2960 | Layer 2 switch, VLANs, trunk to R1 |
| Server-DHCP-DNS | Server | DHCP, DNS and HTTP services |
| PC-Mgmt / PC-Emp / PC-Guest | PC | Test clients |

## Addressing Plan

| VLAN | Name | Network | Gateway |
|---|---|---|---|
| 10 | Management | 192.168.10.0/24 | 192.168.10.1 |
| 20 | Employees | 192.168.20.0/24 | 192.168.20.1 |
| 30 | Guests | 192.168.30.0/24 | 192.168.30.1 |
| 99 | Servers | 192.168.99.0/24 | 192.168.99.1 |

Server static IP: `192.168.99.10`

## Configuration Summary

1. **VLANs on SW1:** VLANs 10, 20, 30, 99 created. Access ports assigned per host, `Fa0/1` configured as a trunk to R1.
2. **Inter-VLAN routing:** sub-interfaces `g0/0.10`, `.20`, `.30`, `.99` with 802.1Q encapsulation on R1.
3. **DHCP:** one pool per VLAN on the server. `ip helper-address 192.168.99.10` on the VLAN 10, 20 and 30 sub-interfaces relays DHCP requests across VLANs.
4. **DNS:** A records for `intranet.company.local` and `web.company.local` pointing to `192.168.99.10`.
5. **Guest isolation ACL** (applied inbound on `g0/0.30`):

```
ip access-list extended GUEST-IN
 permit udp any any eq 67
 permit udp any host 192.168.99.10 eq 53
 deny ip 192.168.30.0 0.0.0.255 192.168.10.0 0.0.0.255
 deny ip 192.168.30.0 0.0.0.255 192.168.20.0 0.0.0.255
 deny ip 192.168.30.0 0.0.0.255 192.168.99.0 0.0.0.255
 permit ip any any
!
interface g0/0.30
 ip access-group GUEST-IN in
```

ACLs are processed top-down and stop at the first match, so DHCP and DNS are permitted first, internal networks are denied next, and all remaining traffic is allowed last.

## Testing and Results

| Test | Expected | Result |
|---|---|---|
| PC-Emp, PC-Mgmt, PC-Guest obtain IPs via DHCP | Correct address, gateway and DNS | Pass |
| `ping intranet.company.local` from PC-Emp | Name resolves, reply received | Pass |
| PC-Emp / PC-Mgmt ping server | Reply received | Pass |
| PC-Guest ping `192.168.99.10` | Blocked | Pass |
| PC-Guest ping `192.168.20.10` | Blocked | Pass |
| PC-Guest ping `192.168.10.10` | Blocked | Pass |
| PC-Guest DHCP and DNS name resolution | Allowed | Pass |

`show access-lists` shows match counters increasing on the deny rules during testing:

![ACL counters](screenshots/show-access-lists.png)

Note: the first ping in a test can time out once because of ARP resolution. This is normal behavior.

## Repository Contents

- `company-network.pkt`: Packet Tracer project file
- `configs/R1-running-config.txt`
- `configs/SW1-running-config.txt`
- `screenshots/`: topology and test results

## Skills Demonstrated

VLANs, 802.1Q trunking, Router-on-a-Stick, DHCP relay, DNS, extended ACLs, Cisco IOS CLI, network testing and documentation.
