# IOS-XE VPN PSK authentication using IKE ID

When using pre-shared key (PSK) authentication on a Site-to-Site VPN using Cisco IOS-XE routers, the IP address of the egress interface is used to match the PSK and authenticate the peer. When spoke routers have a dynamic IP address, which could change, this provides a challenge performing PSK authentication based on the IP address. Some administrators configure a PSK to match “any” IP address (0.0.0.0/0.0.0.0), which is insecure.
One answer is using certificates for authentication, which can create additional complexity, and some administrators shy away from such deployment.

PSK authentication can be used with dynamic authentication based on the IKE identity sent from the spoke, rather than the IP address. On the initiator (the spoke router) the PSK lookup is based on the hub router’s peer IP address or hostname, because the peer IKE identity is not yet known. On responder (the hub router) the PSK lookup can be based on peer IP address or IKE identity, which is received in the IKE_AUTH request from the initiator.

### Key Lookup Parameters

**Peer address**  
Applies to both the initiator and responder.

**Peer hostname**  
Applies only to the initiator when using crypto maps.

**Peer identity type**  
Applies only to the responder.

Supported identity types:
- Address
- Email
- Key-ID
- FQDN

In summary when the spoke router sends its local identity to the hub using either the FQDN, Email or Key-ID, the hub router can use the IKE identity for the PSK lookup. The dynamic public IP address of the spoke router can change and has no impact on authentication to the hub.

# Configuration
The information below represents the relevant configuration to define the IKE identity.

**Hub router**
Create a keyring and define a peer, set the identity to match the identity set in the IKEv2 profile on the peer.

```ruby
crypto ikev2 keyring KEYRING
 peer SPOKE1
  identity fqdn SPOKE1.LAB.LOCAL
  pre-shared-key Cisco1234
 !
 peer SPOKE2
  identity fqdn SPOKE2.LAB.LOCAL
  pre-shared-key Cisco5678
 !
crypto ikev2 profile IKEV2-PROFILE
 match identity remote fqdn domain LAB.LOCAL
 identity local fqdn HUB.LAB.LOCAL
 authentication remote pre-share
 authentication local pre-share
 keyring local KEYRING
 virtual-template 1
```
**Spoke router**  
Spoke must be configured with the IP address of the hub (or 0.0.0.0/0.0.0.0) not the IKE identity.

```ruby
R1 SPOKE
crypto ikev2 keyring KEYRING
 peer HUB
  address 0.0.0.0 0.0.0.0
  pre-shared-key Cisco1234
```
Define the local identity to send to the peer (hub), it’s this identity that is used by the hub to perform the PSK lookup.
```ruby
crypto ikev2 profile IKEV2-PROFILE
 match identity remote fqdn domain LAB.LOCAL
 identity local fqdn SPOKE1.LAB.LOCAL
 authentication remote pre-share
 authentication local pre-share
 keyring local KEYRING
```
# Debugs
The IKE identity used by the hub to match the PSK is identity local fqdn SPOKE1.LAB.LOCAL which is configured on the spoke in the IKEv2 profile.
From Hub
```ruby
 *Apr  3 08:17:36.001: IKEv2:(SESSION ID = 7,SA ID = 1):Searching policy based on peer's identity 'R1.LAB.LOCAL' of type 'FQDN'
*Apr  3 08:17:36.001: IKEv2:found matching IKEv2 profile 'IKEV2-PROFILE'
*Apr  3 08:17:36.001: IKEv2:% Getting preshared key from profile keyring KEYRING
*Apr  3 08:17:36.002: IKEv2:% Matched peer block SPOKE1
*Apr  3 08:17:36.002: IKEv2:Searching Policy with fvrf 0, local address 192.168.251.5
*Apr  3 08:17:36.002: IKEv2:Found Policy 'IKEV2-POLICY'
*Apr  3 08:17:36.003: IKEv2:(SESSION ID = 7,SA ID = 1):not a VPN-SIP session
*Apr  3 08:17:36.003: IKEv2:(SESSION ID = 7,SA ID = 1):Verify peer's policy
*Apr  3 08:17:36.004: IKEv2:(SESSION ID = 7,SA ID = 1):Peer's policy verified
*Apr  3 08:17:36.005: IKEv2:(SESSION ID = 7,SA ID = 1):Get peer's authentication method
*Apr  3 08:17:36.005: IKEv2:(SESSION ID = 7,SA ID = 1):Peer's authentication method is 'PSK'
*Apr  3 08:17:36.005: IKEv2:(SESSION ID = 7,SA ID = 1):Get peer's preshared key for SPOKE1.LAB.LOCAL
*Apr  3 08:17:36.005: IKEv2:(SESSION ID = 7,SA ID = 1):Verify peer's authentication data
*Apr  3 08:17:36.005: IKEv2:(SESSION ID = 7,SA ID = 1):Use preshared key for id SPOKE1.LAB.LOCAL, key len 9
```

