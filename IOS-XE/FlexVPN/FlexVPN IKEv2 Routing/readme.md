# FlexVPN IKEv2 Routing
FlexVPN supports the use of Dynamic Routing protocols such as EIGRP, BGP and OSPF. FlexVPN also has the ability to advertise routes in the IKEv2 SA's. In order to do this we must configured an IKEv2 Authorization Policy, this policy can be configured on the local router or centrally on a RADIUS server such as ISE.

This post describes the steps how to configure a local IKEv2 Authorization Policy on a Hub and Spoke router and advertise routes. The previous blogs post below describe how to set FlexVPN Hub and Spoke and Certificate Authentication.

## Configuration

### Hub
Enable AAA new-model and create a method-list called FLEX_LOCAL using the local database. The FLEX_LOCAL method-list will be referenced in the IKEv2 Profile.
```ruby  
aaa new-model
aaa authorization network FLEX_LOCAL local
```  
Define a loopback address for the Tunnel source.
  
interface Loopback0
description ## FlexVPN Tunnel Source ##
ip address 172.16.0.1 255.255.255.255
  
Create an IKEv2 Authorization Policy, the command route set interface will send the tunnel IP address as a static ip address to the peer. Define each of the subnets to send as an IKEv2 route using the command route set remote ipv4 x.x.x.x
```ruby  
crypto ikev2 authorization policy IKEV2_AUTHZ
  route set interface
  route set remote ipv4 192.168.10.0 255.255.255.0
  route set remote ipv4 192.168.11.0 255.255.255.0
  route set remote ipv4 192.168.251.0 255.255.255.0
```  
Configure aaa authorization and reference the local method-list called FLEX_LOCAL and define the IKEv2 Policy previously configured called IKEV2_AUTHZ.
```ruby  
crypto ikev2 profile IKEV2_PROFILE
  match certificate CERT_MAP
  identity local dn
  authentication remote rsa-sig
  authentication local rsa-sig
  pki trustpoint VPN_TP
  aaa authorization group cert list FLEX_LOCAL IKEV2_AUTHZ
  virtual-template 1 mode auto
```  
### Spoke
Enable AAA new-model and create a method-list called FLEX_LOCAL using the local database. The FLEX_LOCAL method-list will be referenced in the IKEv2 Profile.
```ruby  
aaa new-model
aaa authorization network FLEX_LOCAL local
```  
Define a loopback address for the Tunnel source. We are using Loopback interfaces to simulate local networks on the spoke router.
```ruby  
interface Loopback0
 description ## FlexVPN Tunnel Source ##
 ip address 172.16.1.2 255.255.255.255
!
interface Loopback1
 ip address 10.10.2.1 255.255.255.255
!
interface Loopback2
 ip address 10.10.3.1 255.255.255.255
``` 
Create an standard ACL, define a route for the local subnets to be sent to the peer as a static route. This achieves the same result as using the command route set remote ipv4 x.x.x.x as configured on the Hub router.
```ruby  
ip access-list standard IKEV2_ROUTES
 permit 10.10.2.0 0.0.0.255
 permit 10.10.3.0 0.0.0.255
```  
Create an IKEv2 Authorization Policy, the command route set interface will send the tunnel IP address as a static ip address to the peer. The command router set access-list <ACL_NAME> will send the IP subnets to the peer as IKEv2 routes.
```ruby  
crypto ikev2 authorization policy IKEV2_AUTHZ
 route set interface
 route set access-list IKEV2_ROUTES
```  
Configure aaa authorization and reference the local method-list called FLEX_LOCAL and define the IKEv2 Policy previously configured called IKEV2_AUTHZ.
```ruby  
crypto ikev2 profile IKEV2_PROFILE
 match certificate CERT_MAP
 identity local dn
 authentication local rsa-sig
 authentication remote rsa-sig
 pki trustpoint VPN_TP
 aaa authorization group cert list FLEX_LOCAL IKEV2_AUTHZ
``` 
## Verification
The routing table of the Hub router reveals that the subnets configured in the Authorization Policy (IKEV2_AUTHZ) are learnt from EIGRP. There are other routes learnt from EIGRP, but these are not defined in the Authorization Policy.
  
Run the command show crypto ikev2 sa detailed on the spoke router, notice the section Remote subnets: this list only the subnets learnt via IKEv2 for that peer.



Run the command show crypto ikev2 sa detailed on the Hub router and you will also see the IKEv2 routes learnt from the spoke peer route.
  
 
  
Running the command show ip route will confirm the routes in the Hub's routing table, with the next hop interface as the dynamically created virtual-access interface of the spoke peer.
  
 
  
If additional routes need to be advertised then either the SA's on the receiving route need to be cleared using the command clear crypto session or you must wait until the SA lifetime expires and renegotiated.


