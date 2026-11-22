Enterprise Multi-Router OSPF Network with Centralized DHCP and SSH Security
Platform: EVE-NG | Technology: Cisco IOS/IOSv, OSPF, DHCP relay, SSH
This repository documents an enterprise-style lab with four routers connected in a full mesh, three LAN segments, access switches, VPCS clients, and a server-side network. The design focuses on dynamic routing with OSPF Area 0, centralized DHCP services, DHCP relay, and secure SSH management.
> **Configuration accuracy:** The command blocks below are reconstructed reference templates based on the uploaded topology diagram and project design document. They have not been verified against exports of the actual running configurations. Confirm interface names and addresses in EVE-NG before applying them. Replace the SSH secret placeholder with your own secret. Do not publish real passwords or private keys.
1. EVE-NG topology
![EVE-NG enterprise OSPF topology](enterprise-ospf-centralized-dhcp/topology/enterprise-topology.png)
The topology contains:
Four routers: R1, R2, R3, and R4
Three switches: Switch1, Switch2, and Switch3
Four VPCS clients: VPC1–VPC4
One vIOS node connected to the server-side LAN (intended DHCP server)
Six point-to-point /30 links between the four routers
2. Network addressing plan
Router-to-router links
Link	Network	First endpoint	Second endpoint
R1–R2	`12.1.1.0/30`	R1 Gi0/0 – `12.1.1.1`	R2 Gi0/0 – `12.1.1.2`
R1–R3	`13.1.1.0/30`	R1 Gi0/2 – `13.1.1.1`	R3 Gi0/0 – `13.1.1.2`
R1–R4	`14.1.1.0/30`	R1 Gi0/1 – `14.1.1.1`	R4 Gi0/0 – `14.1.1.2`
R2–R3	`23.1.1.0/30`	R2 Gi0/1 – `23.1.1.1`	R3 Gi0/2 – `23.1.1.2`
R2–R4	`24.1.1.0/30`	R2 Gi0/2 – `24.1.1.1`	R4 Gi0/1 – `24.1.1.2`
R3–R4	`34.1.1.0/30`	R3 Gi0/1 – `34.1.1.1`	R4 Gi0/2 – `34.1.1.2`
LANs and loopbacks
Segment/device	Interface	Address	Purpose
R2 LAN	R2 Gi0/3	`10.1.1.1/24`	Default gateway and DHCP relay for VPC1/VPC2 LAN
R3 LAN	R3 Gi0/3	`20.1.1.1/24`	Default gateway and DHCP relay for VPC3 LAN
R4 server/client LAN	R4 Gi0/3	`30.1.1.1/24`	Gateway for server-side LAN and VPC4
DHCP server node	NIC connected to Switch3	`30.1.1.2/24`	Central DHCP server address in the design notes
R1 Loopback0	—	`1.1.1.1/32`	OSPF router ID; process 100
R2 Loopback0	—	`2.2.2.2/32`	OSPF router ID; process 200
R3 Loopback0	—	`3.3.3.3/32`	OSPF router ID; process 300
R4 Loopback0	—	`4.4.4.4/32`	OSPF router ID; process 400
The process IDs are locally significant; they do not need to match between routers. The topology image labels the physical interfaces. Confirm the interface-to-link mapping on the running nodes before using the templates.
3. Router and DHCP configurations
The full configuration templates are shown below and are also stored as separate text files in `configurations/`.
<details>
<summary><strong>R1 configuration</strong> — interfaces, loopback, OSPF 100 and SSH</summary>
```text
! R1 - reconstructed starter template for the uploaded EVE-NG topology
! Verify interface mapping and replace placeholders before applying.
enable
configure terminal
hostname R1
no ip domain-lookup
!
interface Loopback0
 ip address 1.1.1.1 255.255.255.255
!
interface GigabitEthernet0/0
 description TO-R2-Gi0/0
 ip address 12.1.1.1 255.255.255.252
 no shutdown
!
interface GigabitEthernet0/2
 description TO-R3-Gi0/0
 ip address 13.1.1.1 255.255.255.252
 no shutdown
!
interface GigabitEthernet0/1
 description TO-R4-Gi0/0
 ip address 14.1.1.1 255.255.255.252
 no shutdown
!
router ospf 100
 router-id 1.1.1.1
 network 1.1.1.1 0.0.0.0 area 0
 network 12.1.1.0 0.0.0.3 area 0
 network 13.1.1.0 0.0.0.3 area 0
 network 14.1.1.0 0.0.0.3 area 0
!
! SSH - set a unique secret before using in a real/public environment.
ip domain-name lab.local
username admin privilege 15 secret <SET_A_STRONG_SECRET>
crypto key generate rsa modulus 1024
ip ssh version 2
line vty 0 4
 login local
 transport input ssh
!
end
write memory

```
File: configurations/R1.txt
</details>
<details>
<summary><strong>R2 configuration</strong> — interfaces, OSPF 200, LAN and DHCP relay</summary>
```text
! R2 - reconstructed starter template for the uploaded EVE-NG topology
! Verify interface mapping and replace placeholders before applying.
enable
configure terminal
hostname R2
no ip domain-lookup
!
interface Loopback0
 ip address 2.2.2.2 255.255.255.255
!
interface GigabitEthernet0/0
 description TO-R1-Gi0/0
 ip address 12.1.1.2 255.255.255.252
 no shutdown
!
interface GigabitEthernet0/1
 description TO-R3-Gi0/2
 ip address 23.1.1.1 255.255.255.252
 no shutdown
!
interface GigabitEthernet0/2
 description TO-R4-Gi0/1
 ip address 24.1.1.1 255.255.255.252
 no shutdown
!
interface GigabitEthernet0/3
 description LAN-10.1.1.0-24-TO-SWITCH1
 ip address 10.1.1.1 255.255.255.0
 ip helper-address 30.1.1.2
 no shutdown
!
router ospf 200
 router-id 2.2.2.2
 network 2.2.2.2 0.0.0.0 area 0
 network 12.1.1.0 0.0.0.3 area 0
 network 23.1.1.0 0.0.0.3 area 0
 network 24.1.1.0 0.0.0.3 area 0
 network 10.1.1.0 0.0.0.255 area 0
 passive-interface GigabitEthernet0/3
!
ip domain-name lab.local
username admin privilege 15 secret <SET_A_STRONG_SECRET>
crypto key generate rsa modulus 1024
ip ssh version 2
line vty 0 4
 login local
 transport input ssh
!
end
write memory

```
File: configurations/R2.txt
</details>
<details>
<summary><strong>R3 configuration</strong> — interfaces, OSPF 300, LAN and DHCP relay</summary>
```text
! R3 - reconstructed starter template for the uploaded EVE-NG topology
! Verify interface mapping and replace placeholders before applying.
enable
configure terminal
hostname R3
no ip domain-lookup
!
interface Loopback0
 ip address 3.3.3.3 255.255.255.255
!
interface GigabitEthernet0/0
 description TO-R1-Gi0/2
 ip address 13.1.1.2 255.255.255.252
 no shutdown
!
interface GigabitEthernet0/2
 description TO-R2-Gi0/1
 ip address 23.1.1.2 255.255.255.252
 no shutdown
!
interface GigabitEthernet0/1
 description TO-R4-Gi0/2
 ip address 34.1.1.1 255.255.255.252
 no shutdown
!
interface GigabitEthernet0/3
 description LAN-20.1.1.0-24-TO-SWITCH2
 ip address 20.1.1.1 255.255.255.0
 ip helper-address 30.1.1.2
 no shutdown
!
router ospf 300
 router-id 3.3.3.3
 network 3.3.3.3 0.0.0.0 area 0
 network 13.1.1.0 0.0.0.3 area 0
 network 23.1.1.0 0.0.0.3 area 0
 network 34.1.1.0 0.0.0.3 area 0
 network 20.1.1.0 0.0.0.255 area 0
 passive-interface GigabitEthernet0/3
!
ip domain-name lab.local
username admin privilege 15 secret <SET_A_STRONG_SECRET>
crypto key generate rsa modulus 1024
ip ssh version 2
line vty 0 4
 login local
 transport input ssh
!
end
write memory

```
File: configurations/R3.txt
</details>
<details>
<summary><strong>R4 configuration</strong> — interfaces, OSPF 400 and server-side LAN</summary>
```text
! R4 - reconstructed starter template for the uploaded EVE-NG topology
! Verify interface mapping and replace placeholders before applying.
enable
configure terminal
hostname R4
no ip domain-lookup
!
interface Loopback0
 ip address 4.4.4.4 255.255.255.255
!
interface GigabitEthernet0/0
 description TO-R1-Gi0/1
 ip address 14.1.1.2 255.255.255.252
 no shutdown
!
interface GigabitEthernet0/1
 description TO-R2-Gi0/2
 ip address 24.1.1.2 255.255.255.252
 no shutdown
!
interface GigabitEthernet0/2
 description TO-R3-Gi0/1
 ip address 34.1.1.2 255.255.255.252
 no shutdown
!
interface GigabitEthernet0/3
 description SERVER-AND-CLIENT-LAN-30.1.1.0-24-TO-SWITCH3
 ip address 30.1.1.1 255.255.255.0
 no shutdown
!
router ospf 400
 router-id 4.4.4.4
 network 4.4.4.4 0.0.0.0 area 0
 network 14.1.1.0 0.0.0.3 area 0
 network 24.1.1.0 0.0.0.3 area 0
 network 34.1.1.0 0.0.0.3 area 0
 network 30.1.1.0 0.0.0.255 area 0
 passive-interface GigabitEthernet0/3
!
ip domain-name lab.local
username admin privilege 15 secret <SET_A_STRONG_SECRET>
crypto key generate rsa modulus 1024
ip ssh version 2
line vty 0 4
 login local
 transport input ssh
!
end
write memory

```
File: configurations/R4.txt
</details>
<details>
<summary><strong>DHCP server configuration example</strong> — server address and DHCP pools</summary>
```text
! Example only: use this if the vIOS node in the topology is acting as
! a Cisco IOS DHCP server. If it runs Linux/another OS, configure its
! DHCP service there instead. Confirm the server-facing interface first.
!
! Server-side address (from project design notes): 30.1.1.2/24
! Default gateway: 30.1.1.1
!
enable
configure terminal
! Replace GigabitEthernet0/0 with the interface connected to Switch3.
interface GigabitEthernet0/0
 description TO-SWITCH3-SERVER-LAN
 ip address 30.1.1.2 255.255.255.0
 no shutdown
!
ip dhcp excluded-address 10.1.1.1 10.1.1.9
ip dhcp excluded-address 20.1.1.1 20.1.1.9
ip dhcp excluded-address 30.1.1.1 30.1.1.9
!
ip dhcp pool POOL_10
 network 10.1.1.0 255.255.255.0
 default-router 10.1.1.1
 dns-server 8.8.8.8
!
ip dhcp pool POOL_20
 network 20.1.1.0 255.255.255.0
 default-router 20.1.1.1
 dns-server 8.8.8.8
!
! Optional pool for VPC4 on the server-side LAN.
ip dhcp pool POOL_30
 network 30.1.1.0 255.255.255.0
 default-router 30.1.1.1
 dns-server 8.8.8.8
!
end
write memory

```
File: configurations/DHCP-SERVER.txt
Important: This is an IOS DHCP-server example. If the vIOS node in the lab is not acting as a Cisco IOS DHCP server, configure the DHCP service according to the operating system actually running on that node.
</details>
4. How the network works
OSPF
All four routers participate in OSPF Area 0. Each router uses its loopback address as its router ID. The six /30 links provide multiple paths between routers. LAN-facing interfaces are configured as passive in the reference templates so the LAN networks can be advertised without sending OSPF hello packets toward end devices.
Centralized DHCP and relay
The design places the DHCP server at `30.1.1.2/24` on the R4-side LAN. DHCP discovery messages from remote LAN clients are broadcasts and do not cross routers by default. The R2 LAN interface and R3 LAN interface therefore use `ip helper-address 30.1.1.2` to relay client requests to the central server. R4's directly connected server LAN does not need a relay for the server itself.
SSH management
The router templates include a domain name, local privileged user placeholder, RSA key generation, SSH version 2, local VTY authentication, and `transport input ssh`. Replace `<SET_A_STRONG_SECRET>` with a unique secret before applying. Never commit actual credentials or private keys to a public repository.
5. Verification commands
Run these on the relevant Cisco routers and save the actual outputs/screenshots in the `verification/` folders:
Purpose	Command
Interface status and addressing	`show ip interface brief`
OSPF neighbors	`show ip ospf neighbor`
OSPF-learned routes	`show ip route ospf`
OSPF process details	`show ip protocols`
DHCP bindings (on IOS DHCP server)	`show ip dhcp binding`
SSH status	`show ip ssh`
RSA public key details	`show crypto key mypubkey rsa`
From VPCS clients, check assigned addressing with `show ip`, then test reachability using `ping` and `trace` as supported by VPCS. Test between client LANs and to the DHCP server address `30.1.1.2`. Verify SSH access between routers and confirm Telnet is not permitted if SSH-only VTY access is configured.
> Add your real command outputs and screenshots before describing the lab as fully verified. Do not claim successful neighbor formation, DHCP leases, or end-to-end pings unless you have captured those results.
6. Troubleshooting notes
OSPF adjacency does not form: Check interface status, IP addresses/masks, Area 0 configuration, router IDs, and passive-interface settings.
DHCP clients do not receive leases: Check the DHCP service and pools, excluded addresses, routing to `30.1.1.2`, and `ip helper-address` on the R2/R3 client-facing interfaces.
SSH does not start: Configure the domain name before generating RSA keys; check SSH version, local user, and VTY login settings.
A /30 link does not communicate: Confirm both endpoints use addresses in the same /30 subnet and that both interfaces are enabled.

