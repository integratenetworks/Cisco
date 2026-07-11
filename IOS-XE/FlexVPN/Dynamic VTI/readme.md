# Cisco FlexVPN Dynamic VTI (Hub-and-Spoke)
In a FlexVPN Hub and Spoke design spoke routers are configured with a normal static VTI with the tunnel destination of the Hub’s IP address, the Hub however is configured with a Dynamic VTI. The DVTI on the Hub router is not configured with a static mapping to the peer’s IP address. The VTI on the Hub is created dynamically from a preconfigured tunnel template “virtual-template” when a tunnel is initiated by the spoke router/peer. The dynamic tunnel spawns a separate “virtual-access” interface for each spoke tunnel, inheriting the configuration from the cloned the template.
 

## Configuration

### Hub Router

Define a loopback interface (this will be used as a source IP address for the tunnel)
```ruby
interface loopback 0
 ip address 172.16.0.1 255.255.255.255
```
Create a Tunnel Template (tunnel of source WAN interface and use Lo0 as IP for Tunnel) 
```ruby
interface virtual-template 1 type tunnel
 tunnel source gigabitethernet 0/0 
 ip unnumbered loopback 0
```
Create a PSK Keyring (use address of 0.0.0.0 for lab purposes to match all peers, use symmetric PSK key for simplicity)
```ruby
crypto ikev2 keyring KEYRING
 peer ANY-PEER
  address 0.0.0.0
  pre-shared-key local cisco1234
  pre-shared-key remote cisco1234
  exit
```
Create IKEv2 Profile (specify local identity of FQDN, match any peer on the domain name, specify authentication PSK, specify the Keyring to use and specify the Virtual Template to clone)  
```ruby
crypto ikev2 profile IKEV2_PROFILE
 match identity remote fqdn domain lab.net
 identity local fqdn R1.lab.net
 authentication remote pre-share
 authentication local pre-share
 keyring local KEYRING
 virtual-template 1
```
Create IPSec Profile (set IKEv2 Profile, default Transform set will be used so no need to specify)
```ruby
crypto ipsec profile IPSEC_PROFILE
 set ikev2-profile IKEV2_PROFILE
```
Specify the IPSec Profile on the Tunnel Template
```ruby
interface virtual-template 1 type tunnel
 tunnel protection ipsec profile IPSEC_PROFILE
```
Specify some Loopback Interfaces to simulate LAN Subnets & configure Dynamic Routing Protocol
```ruby
interface loopback1
 ip address 10.1.0.1 255.255.255.0
!
interface loopback2
 ip address 10.1.1.1 255.255.255.0
!
router eigrp 1
 no auto-summary
 network 172.16.0.1
 network 10.1.0.0 0.0.255.255
```
### Spoke Router
Step 1 - Specify some Loopback Interfaces to simulate LAN Subnets & configure Dynamic Routing Protocol
interface loopback0
 ip address 172.16.0.2 255.255.255.0
interface loopback1
 ip address 10.3.0.1 255.255.255.0
interface loopback2
 ip address 10.3.1.1 255.255.255.0 
router eigrp 1
 no auto-summary
 network 172.16.0.2
 network 10.3.0.0 0.0.255.255

Step 2 – Create a PSK Keyring (use address of 0.0.0.0 for lab purposes to match all peers, use symmetric PSK key for simplicity)
crypto ikev2 keyring KEYRING
 peer ANY-PEER
  address 0.0.0.0
  pre-shared-key local cisco1234
  pre-shared-key remote cisco1234
  exit

Step 3 – Create IKEv2 Profile (specify local identity of FQDN, match any peer on the domain name, specify authentication PSK, specify the Keyring to use)
crypto ikev2 profile IKEV2_PROFILE
 match identity remote fqdn domain lab.net
 identity local fqdn R2.lab.net
 authentication remote pre-share
 authentication local pre-share
 keyring local KEYRING

Step 4 – Create IPSec Profile (set IKEv2 Profile, default Transform set will be used so no need to specify)
crypto ipsec profile IPSEC_PROFILE
 set ikev2-profile IKEV2_PROFILE
 
Step 5 – Create a SVTI (use Lo0 as tunnel interface, specify tunnel source, tunnel destination as Hub’s WAN IP, specify IPSec Profile) 
interface tunnel0
 ip unnumbered loopback 0
 tunnel source gigabitethernet 0/0
 tunnel destination 1.1.1.1
 tunnel protection ipsec profile IPSEC_PROFILE

Verify Configuration
Use the command “show ip interface brief” to display that a virtual-access interface has been created.
 
Using the command “show crypto ikev2 sa detailed” you can verify the IKEv2 SA was established correct with the peer
 
Use the “show crypto ipsec sa” command to configure the IPSec tunnel is UP and passing traffic. Each IPSec SA will identify the Virtual-Access interface associated with the remote ID of the peer.
 


