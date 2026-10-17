# Multi-Site Enterprise Network with OSPF, NAT and ACLs

A two-site enterprise network built and tested in **Cisco Packet Tracer**. The Head Office (HQ) and Branch Office are connected with OSPF, all users share one public IP through PAT, and department access is controlled with extended ACLs.

> Connect two offices into one network, give everyone internet access through a single gateway, and control who can talk to whom.

## Overview

| Item | Detail |
|---|---|
| Tool | Cisco Packet Tracer |
| Sites | HQ and Branch |
| Routing | OSPF process 1, area 0 |
| Internet access | NAT with PAT (overload) on HQ router |
| Segmentation | VLANs with router-on-a-stick |
| Security | Extended named ACLs (inbound on VLAN subinterfaces) |

## Use Case

A small company has opened a second office. The network must satisfy these requirements:

- Branch employees can reach HQ resources as if on the same network.
- All staff have internet access through **one public IP** and **one controlled exit point**.
- Departments are separated: Sales cannot browse the IT network, and Accounts cannot roam the internal network.
- IT staff can still manage and troubleshoot every department.

**HQ departments:** Sales, IT, Servers
**Branch departments:** Sales, Accounts

## Problem Statements and Solutions

| # | Problem | Solution |
|---|---|---|
| 1 | The two offices are isolated; routers only know directly attached networks | **OSPF area 0** exchanges routes dynamically. It scales better than static routes and adapts to topology changes |
| 2 | Private addresses (10.x.x.x) are not routable on the internet and the company has one public IP | **NAT with PAT (overload)** translates all internal hosts to the HQ public interface address |
| 3 | A flat network lets any device reach any other, which is a security risk | **Extended ACLs** enforce least privilege per department |
| 4 | Separate hardware per department is costly and inflexible | **VLANs** plus **router-on-a-stick** segment departments on shared hardware |

## Topology

                          [Internet-Server 8.8.8.8]
                                    |  8.8.8.0/24
                                [  ISP  ]
                                    |  203.0.113.0/30   (NAT outside)
   HQ LANs                      [ HQ-R1 ] ---- 10.0.0.0/30 ---- [ BR-R1 ]     Branch LANs
 VLAN10 Sales   10.1.10.0/24        |                                |        VLAN10 Sales    10.2.10.0/24
 VLAN20 IT      10.1.20.0/24    [ SW-HQ ]                        [ SW-BR ]    VLAN20 Accounts 10.2.20.0/24
 VLAN30 Servers 10.1.30.0/24

## Devices

| Device | Model | Role |
|---|---|---|
| HQ-R1 | Cisco 2911 | HQ gateway, inter-VLAN routing, OSPF, NAT, ACLs |
| BR-R1 | Cisco 2911 | Branch gateway, inter-VLAN routing, OSPF, ACLs |
| ISP | Cisco 2911 | Simulated internet provider |
| SW-HQ | Cisco 2960 | HQ VLAN switch |
| SW-BR | Cisco 2960 | Branch VLAN switch |
| HQ-Server | Server-PT | Internal web server |
| Internet-Server | Server-PT | Public web server |
| PC-HQ-Sales, PC-HQ-IT, PC-BR-Sales, PC-BR-Acct | PC-PT | End users |

## IP Addressing Plan

| Device | Interface | IP / Mask | Notes |
|---|---|---|---|
| ISP | Gi0/0 | 203.0.113.1/30 | to HQ |
| ISP | Gi0/1 | 8.8.8.1/24 | to Internet-Server |
| HQ-R1 | Gi0/2 | 203.0.113.2/30 | NAT outside |
| HQ-R1 | Gi0/1 | 10.0.0.1/30 | to Branch |
| HQ-R1 | Gi0/0.10 | 10.1.10.1/24 | Sales gateway |
| HQ-R1 | Gi0/0.20 | 10.1.20.1/24 | IT gateway |
| HQ-R1 | Gi0/0.30 | 10.1.30.1/24 | Servers gateway |
| BR-R1 | Gi0/1 | 10.0.0.2/30 | to HQ |
| BR-R1 | Gi0/0.10 | 10.2.10.1/24 | Sales gateway |
| BR-R1 | Gi0/0.20 | 10.2.20.1/24 | Accounts gateway |
| Internet-Server | NIC | 8.8.8.8/24 | GW 8.8.8.1 |
| HQ-Server | NIC | 10.1.30.10/24 | GW 10.1.30.1 |
| PC-HQ-Sales | NIC | 10.1.10.10/24 | GW 10.1.10.1 |
| PC-HQ-IT | NIC | 10.1.20.10/24 | GW 10.1.20.1 |
| PC-BR-Sales | NIC | 10.2.10.10/24 | GW 10.2.10.1 |
| PC-BR-Acct | NIC | 10.2.20.10/24 | GW 10.2.20.1 |

**VLANs:** HQ: 10 Sales, 20 IT, 30 Servers. Branch: 10 Sales, 20 Accounts.

## Security Policy

| Group | Allowed | Blocked |
|---|---|---|
| IT (HQ) | Everything | Nothing |
| Sales (HQ and Branch) | Web (TCP 80) to HQ-Server, internet, other sites | Other server traffic, IT subnet |
| Accounts (Branch) | Web (TCP 80) to HQ-Server, internet | All other internal 10.0.0.0/8 networks |

## Configuration Summary

### Switches
- VLANs created and named; access ports assigned per department.
- Uplink to the router configured as an 802.1Q trunk (`switchport mode trunk`).

### Routers (router-on-a-stick)
Each router uses a physical LAN interface with dot1Q subinterfaces, one per VLAN, each carrying the gateway IP.

### OSPF
router ospf 1
 router-id <1.1.1.1 on HQ-R1 | 2.2.2.2 on BR-R1>
 passive-interface default
 no passive-interface gi0/1
 network <each LAN and the 10.0.0.0/30 link> area 0
 default-information originate      ! HQ-R1 only
 
- LAN-facing interfaces are passive, so no OSPF hellos are sent to end users.
- HQ-R1 has a static default route to the ISP and advertises it to the Branch.

### NAT / PAT (HQ-R1)
access-list 1 permit 10.0.0.0 0.255.255.255
ip nat inside source list 1 interface gi0/2 overload

LAN subinterfaces and `gi0/1` are `ip nat inside`; `gi0/2` is `ip nat outside`. Inter-site traffic never crosses an inside-to-outside boundary, so it is not translated.

### ACLs
Applied **inbound** on each department's VLAN subinterface. Rule order matters because the first match wins.

ip access-list extended HQ-SALES-IN
 permit icmp 10.1.10.0 0.0.0.255 10.1.20.0 0.0.0.255 echo-reply
 permit tcp 10.1.10.0 0.0.0.255 host 10.1.30.10 eq 80
 deny ip 10.1.10.0 0.0.0.255 10.1.30.0 0.0.0.255
 deny ip 10.1.10.0 0.0.0.255 10.1.20.0 0.0.0.255
 permit ip any any
 
`BR-SALES-IN` mirrors this for the Branch Sales subnet. `BR-ACCT-IN` permits web to the server, then denies all of `10.0.0.0/8`, then permits the rest (internet).

> Full device configurations are in the `configs/` folder.

## Test Plan and Results

Web tests use `http://10.1.30.10` from the PC browser.

| # | Test | Expected | Result |
|---|---|---|---|
| 1 | HQ-Sales to BR-Sales ping | Success | |
| 2 | HQ-Sales to HQ-IT ping | Blocked | |
| 3 | HQ-IT to HQ-Sales ping | Success | |
| 4 | HQ-Sales to HQ-Server HTTP | Success | |
| 5 | HQ-Sales to HQ-Server ping | Blocked | |
| 6 | HQ-IT to HQ-Server ping and HTTP | Success | |
| 7 | BR-Sales to HQ-Server HTTP | Success | |
| 8 | BR-Sales to HQ-IT ping | Blocked | |
| 9 | BR-Acct to BR-Sales ping | Blocked | |
| 10 | BR-Acct to HQ-Server HTTP | Success | |
| 11 | BR-Acct to 8.8.8.8 ping and HTTP | Success | |
| 12 | Any PC to 8.8.8.8 | Success, visible in NAT table | |

Some "blocked" tests show a timeout instead of "destination unreachable" when it is the reply that gets dropped. Both mean the policy works.

## Verification Commands

show ip ospf neighbor          ! neighbor in FULL state
show ip route                  ! O and O*E2 routes present
show ip nat translations       ! inside local 10.x mapped to 203.0.113.2
show ip nat statistics
show access-lists              ! match counters increase on blocked traffic
show vlan brief                ! on switches
show interfaces trunk          ! on switches

## Troubleshooting Notes

| Symptom | Likely cause |
|---|---|
| No OSPF neighbor | Gi0/1 still passive, wrong `network` statement, interface shut down |
| PC cannot reach its gateway | Trunk down, wrong access VLAN, subinterface VLAN ID mismatch |
| Branch has no internet | `default-information originate` missing, or Gi0/1 not `ip nat inside` |
| Empty NAT table | Inside/outside roles wrong, or ACL 1 does not match the source |
| ACL blocks everything | Missing `permit ip any any`, or wrong direction |
| First ping times out | Normal ARP delay |

## Key Learnings

- **ACLs are stateless.** A rule that blocks Sales from reaching IT also blocks IT's pings from getting replies back, unless echo-reply is explicitly permitted.
- **ACL order is critical.** Specific permits must come before broad denies.
- **Passive interfaces** stop OSPF hellos on user-facing ports, which is cleaner and safer.
- **NAT only applies across inside/outside boundaries**, so site-to-site traffic keeps its real addresses.
- Verify each layer in order: VLANs, then routing, then NAT, then ACLs. Adding security last makes faults far easier to isolate.

## Scope and Limitations

- This is a simulation; the "internet" is an ISP router plus one server.
- Single OSPF area and static host IPs, for clarity.
- Security is at Layer 3/4 only (ACLs). There is no firewall, VPN or authentication.
- The Branch has a single WAN link, so there is no redundancy.

## Future Improvements

- DHCP pools per VLAN
- Port security on access ports
- SSH-only management with local or AAA authentication
- Second Branch-to-ISP link with a floating static route
- Site-to-site VPN between HQ and Branch
- Stateful firewall (zone-based or ASA) in place of ACLs

## Skills Demonstrated

IP planning and subnetting, VLANs and trunking, inter-VLAN routing, OSPF, NAT/PAT, ACL design, network verification and troubleshooting, technical documentation.

## Repository Structure
.
├── README.md
├── MultiSite-OSPF-NAT-ACL.pkt
├── configs/
│   ├── HQ-R1.txt
│   ├── BR-R1.txt
│   ├── ISP.txt
│   ├── SW-HQ.txt
│   └── SW-BR.txt
└── images/
    ├── topology.png
    └── test-results/

## Author

Pavith R· B.Tech IT (2026) · Aspiring Network Engineer
