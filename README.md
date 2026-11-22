# Enterprise Multi-Router OSPF Network with Centralized DHCP and SSH

A Cisco IOS networking lab built in **EVE-NG** to practise dynamic routing, centralized IP address assignment, DHCP relay, and secure device management.

> **Repository accuracy note:** The topology image uploaded for this repository shows the lab layout and interface labels. The accompanying project document contains the addressing plan, requirements, and example command snippets, but it does not contain complete running configurations or captured verification outputs. Configuration files in `configurations/` are therefore **reconstructed starter templates based on those materials**, not verified exports from the devices. Replace or update them with your actual running configurations before presenting them as the exact commands you executed.

## Project objectives

- Build a four-router full-mesh topology (R1–R4).
- Configure OSPF Area 0 using loopbacks as stable router IDs.
- Advertise the inter-router point-to-point networks and LAN networks.
- Provide DHCP centrally from the server segment and relay client requests across routers using `ip helper-address`.
- Configure SSH v2 and local authentication for secure remote management; restrict VTY access to SSH.
- Verify OSPF adjacencies, learned routes, DHCP leases, and end-to-end connectivity.

## Tools and technologies

- EVE-NG
- Cisco IOS / IOSv
- OSPF (single Area 0)
- IPv4 addressing and /30 point-to-point links
- DHCP and DHCP relay (`ip helper-address`)
- SSH v2, RSA keys, local user authentication
- VPCS clients

## Topology

<img width="1543" height="857" alt="Screenshot 2026-11-22 122027" src="https://github.com/user-attachments/assets/e636b128-9813-41cd-8875-de562daa4be7" />

The uploaded EVE-NG diagram contains:

- Four routers: R1, R2, R3, and R4
- Three access switches: Switch1, Switch2, and Switch3
- Four VPCS endpoints: VPC1–VPC4
- One vIOS node shown on the server-side LAN (intended DHCP-server node)
- Six point-to-point router links forming a full mesh

### Interface and link map from the uploaded diagram

| Link | Interfaces shown | IPv4 network |
|---|---|---|
| R1–R2 | R1 Gi0/0 ↔ R2 Gi0/0 | 12.1.1.0/30 |
| R1–R3 | R1 Gi0/2 ↔ R3 Gi0/0 | 13.1.1.0/30 |
| R1–R4 | R1 Gi0/1 ↔ R4 Gi0/0 | 14.1.1.0/30 |
| R2–R3 | R2 Gi0/1 ↔ R3 Gi0/2 | 23.1.1.0/30 |
| R2–R4 | R2 Gi0/2 ↔ R4 Gi0/1 | 24.1.1.0/30 |
| R3–R4 | R3 Gi0/1 ↔ R4 Gi0/2 | 34.1.1.0/30 |
| R2 LAN | R2 Gi0/3 ↔ Switch1 | 10.1.1.0/24 |
| R3 LAN | R3 Gi0/3 ↔ Switch2 | 20.1.1.0/24 |
| R4 server/client LAN | R4 Gi0/3 ↔ Switch3 | 30.1.1.0/24 |

The interface mapping above is read from the uploaded diagram. Confirm it with `show cdp neighbors` / `show ip interface brief` in EVE-NG before applying configurations.

## IP addressing plan

### Router-to-router links

| Network | Router/interface | Address |
|---|---|---|
| 12.1.1.0/30 | R1 Gi0/0 | 12.1.1.1/30 |
|  | R2 Gi0/0 | 12.1.1.2/30 |
| 13.1.1.0/30 | R1 Gi0/2 | 13.1.1.1/30 |
|  | R3 Gi0/0 | 13.1.1.2/30 |
| 14.1.1.0/30 | R1 Gi0/1 | 14.1.1.1/30 |
|  | R4 Gi0/0 | 14.1.1.2/30 |
| 23.1.1.0/30 | R2 Gi0/1 | 23.1.1.1/30 |
|  | R3 Gi0/2 | 23.1.1.2/30 |
| 24.1.1.0/30 | R2 Gi0/2 | 24.1.1.1/30 |
|  | R4 Gi0/1 | 24.1.1.2/30 |
| 34.1.1.0/30 | R3 Gi0/1 | 34.1.1.1/30 |
|  | R4 Gi0/2 | 34.1.1.2/30 |

### LANs and loopbacks

| Device | Interface | Address | Purpose |
|---|---|---|---|
| R2 | Gi0/3 | 10.1.1.1/24 | LAN gateway; DHCP relay |
| R3 | Gi0/3 | 20.1.1.1/24 | LAN gateway; DHCP relay |
| R4 | Gi0/3 | 30.1.1.1/24 | Server/client LAN gateway |
| DHCP server node | NIC connected to Switch3 | 30.1.1.2/24 | Central DHCP service (as specified in project notes) |
| R1 | Loopback0 | 1.1.1.1/32 | OSPF router ID; process 100 |
| R2 | Loopback0 | 2.2.2.2/32 | OSPF router ID; process 200 |
| R3 | Loopback0 | 3.3.3.3/32 | OSPF router ID; process 300 |
| R4 | Loopback0 | 4.4.4.4/32 | OSPF router ID; process 400 |

The original project notes specify DHCP pools for `10.1.1.0/24` and `20.1.1.0/24`, and an optional pool for `30.1.1.0/24`. The exact VPCS-to-switch port assignments and the DHCP server operating mode should be confirmed in the lab.

## Configuration files

| File | Contents |
|---|---|
| `configurations/R1.txt`–`R4.txt` | Router interface, loopback, OSPF, and SSH starter configurations |
| `configurations/DHCP-SERVER.txt` | IOS DHCP-server example for the server-side vIOS node; adapt if the node runs a different server OS/service |

**Before applying:** confirm interface names, link endpoints, and the DHCP server type. Set your own secure local username/secret. Do not commit real passwords, private keys, or other credentials to a public repository.

## DHCP design

- DHCP server address: `30.1.1.2/24`
- Default gateway on server LAN: `30.1.1.1`
- Relay target on remote LANs: `30.1.1.2`
- R2 relays DHCP requests received on its `10.1.1.0/24` LAN.
- R3 relays DHCP requests received on its `20.1.1.0/24` LAN.
- R4's server-side LAN does not need a relay because the server is directly connected to that subnet.

## OSPF design

All four routers participate in **Area 0**. The process IDs (100, 200, 300, and 400) are locally significant; they do not need to match. Each router uses its loopback address as its OSPF router ID. LAN-facing interfaces should be passive while still advertising their connected networks.

## Verification checklist

Save your real command output or screenshots in the matching `verification/` folder. The folders are intentionally empty until you add evidence from your own EVE-NG lab.

```text
show ip interface brief
show ip ospf neighbor
show ip route ospf
show ip protocols
show ip dhcp binding
show ip ssh
show crypto key mypubkey rsa
```

From the VPCS clients, verify DHCP assignment and test connectivity with `show ip`, `ping`, and `trace` (VPCS command syntax may differ from Windows `ipconfig` / `tracert`). Test reachability between client subnets and to `30.1.1.2`. Confirm SSH access works and Telnet is not accepted on the configured VTY lines.

## Repository layout

```text
enterprise-ospf-centralized-dhcp/
├── README.md
├── topology/
│   └── enterprise-topology.png
├── configurations/
    ├── R1.txt
    ├── R2.txt
    ├── R3.txt
    ├── R4.txt
    └── DHCP-SERVER.txt

```

## Troubleshooting notes

- OSPF adjacency does not form: check interface state, addressing/masks, area, and whether the interface was accidentally made passive.
- DHCP clients do not receive leases: verify the DHCP pool, excluded addresses, routing to `30.1.1.2`, and `ip helper-address` on R2/R3 LAN interfaces.
- SSH does not start: configure the domain name before generating RSA keys, use SSH v2, and check local login/VTY settings.
- A /30 link does not communicate: confirm both endpoints are in the same /30 subnet and interfaces are enabled.

## Conclusion
This project documents the design of an enterprise-style network in EVE-NG using four interconnected routers, OSPF Area 0, loopback interfaces, three LAN segments, and a centralized DHCP service with relay configuration. It also covers SSH-based remote management to demonstrate basic network-device security practices.

The lab brings together important CCNA-level networking concepts, including IPv4 addressing and subnetting, point-to-point links, dynamic routing, DHCP relay, and secure device administration.


