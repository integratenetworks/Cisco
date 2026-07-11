# Cisco FlexVPN Static VTI (sVTI)
This post provides a simple configuration example when using Smart Defaults and when using custom configurations.

## Configuration - FlexVPN SVTI with Smart Defaults
This simple lab configuration is to setup a SVTI Site-to-Site VPN between 2 Cisco IOS routers.
 
Define the WAN interface, loopback and dynamic routing protocol
```ruby
interface gigabitethernet 0/0
 ip address 1.1.1.1 255.255.255.0
!
interface loopback 0
 ip address 172.16.0.1 255.255.255.0
!
router eigrp 1
 no auto-summary
 network 172.16.0.1
```
Create the IKEv2 Keyring, specify the peer’s WAN IP address and PSKs
```ruby
crypto ikev2 keyring KEYRING
 peer R2
  address 1.1.1.2
  pre-shared-key local cisco1234
  pre-shared-key remote cisco5678
```
Create the IKEv2 Profile, match the identity of the peer router, specify the local router’s identity, specify authentication method and reference the local IKEv2 Keyring
```ruby
crypto ikev2 profile IKEV2_PROFILE
 match identity remote fqdn domain lab.net
 identity local fqdn R1.lab.net 
 authentication local pre-share
 authentication remote pre-share
 keyring local KEYRING
```
Modify the default IPSec Profile to reference the newly created IKEv2 Profile
```ruby
crypto ipsec profile default
 set ikev2-profile IKEV2_PROFILE
```
Create a tunnel interface, specify the interface Lo0 as the source IP address for the tunnel interface, tunnel source interface, tunnel destination, tunnel mode and the IPSec profile
```ruby
interface tunnel 0
 ip unnumbered loopback 0
 tunnel source gigabitethernet 0/0
 tunnel destination 1.1.1.2
 tunnel mode ipsec ipv4
 tunnel protection ipsec profile default
```
The router specific settings will obviously need to be modified for the configuration of the peer router (Keyring peer, Keyring address, IKEv2 Profile local identity, IP addresses etc).

## Configuration - FlexVPN SVTI without Smart Defaults
As in the example above the scenario is exactly the same, but this time with Smart Defaults disabled and custom IKEv2 Proposal, Policy, Profile and IPSec Transform Set, Profile.

Define the WAN interface, loopback and dynamic routing protocol
```ruby
interface gigabitethernet 0/0
 ip address 1.1.1.1 255.255.255.0
!
interface loopback 0
 ip address 172.16.0.1 255.255.255.0
!
router eigrp 1
 no auto-summary
 network 172.16.0.1
```
Disable the Smart Defaults
```ruby
no crypto ikev2 proposal default
no crypto ikev2 policy default
no crypto ipsec profile default
no crypto ipsec transform-set default
```
Create an IKEv2 Proposal
```ruby
crypto ikev2 proposal PROPOSAL-1
 encryption aes-cbc-256
 group 14
 integrity sha256
```
Create IKEv2 Policy and reference the previously created IKEv2 Proposal
```ruby
crypto ikev2 policy IKEV2_POLICY
 proposal PROPOSAL-1
```
Create the IKEv2 Keyring, specify the peer’s WAN IP address and PSKs
```ruby
crypto ikev2 keyring KEYRING
 peer R2
  address 1.1.1.2
  pre-shared-key local cisco1234
  pre-shared-key remote cisco5678
```
Create the IKEv2 Profile, match the identity of the peer router, specify the local router’s identity, specify authentication method and reference the local IKEv2 Keyring
```ruby
crypto ikev2 profile IKEV2_PROFILE
 match identity remote fqdn domain lab.net
 identity local fqdn R1.lab.net 
 authentication local pre-share
 authentication remote pre-share
 keyring local KEYRING
```
Create a new IPSec Transform Set
```ruby
crypto ipsec transform-set TSET esp-aes 256 esp-sha256-hmac
```
Create a new IPSec Profile, reference the newly created Transform Set and IKEv2 Profile
```ruby
crypto ipsec profile IPSEC_PROFILE
 set ikev2-profile IKEV2_PROFILE
 set ipsec transform-set TSET
```
Create a tunnel interface, specify the interface Lo0 as the source IP address for the tunnel interface, tunnel source interface, tunnel destination, tunnel mode and the IPSec profile
```ruby
interface tunnel 0
 ip unnumbered loopback 0
 tunnel source gigabitethernet 0/0
 tunnel destination 1.1.1.2
 tunnel mode ipsec ipv4
 tunnel protection ipsec profile IPSEC_PROFILE
```
As before router specific settings will obviously need to be modified for the configuration of the peer router (Keyring peer, Keyring address, IKEv2 Profile local identity, IP addresses etc).

# Verify Commands

Confirm IKEv2 SA “show crypto ikev2 sa”. If IKEv2 SA has completed successfully you should see the SAs
 
Confirm IPSec SA “show crypto ipsec sa”. If IPSec SA has established correctly you should see pkts encaps/decaps increase and traffic pass over the VPN.
  Confirm IPSec Profile Settings “show crypto ipsec profile”. This will list all IPSec profiles and what IKEv2 Profile and Transform Set has been referenced within the IPSec Profile
 

Confirm IKEv2 Proposal “show crypto ikev2 proposal”. This will list all proposals and the settings configured
 

