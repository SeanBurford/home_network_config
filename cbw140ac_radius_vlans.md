# Identity Networking on Cisco Business Wireless APs

This page describes how to use Radius to assign VLANs to clients on Cisco
Business Wifi 140ac/240ac Access Points.  Under the hood, these access points
appear to include the Cisco Wireless LAN Controller software, which usually runs
on a separate server (but here it runs on the APs).

Dynamically assigning devices to VLANs allows you to tailor network access and
services to different devices without setting up multiple networks/SSIDs.
This is part of what Cisco calls Cisco Identity-Based Networking, which
encompasses authentication, access control, and user policy enforcement.

## AP Configuration

In order to assign VLANs to clients based on identity, you have to:

*  Configure the CBW APs with the VLANs.
*  Enable both "WPA2 Enterprise" and "AAA Override" on each WLAN.

### Background: Upstream RLAN 

Existing WLANs on my APs were configured to tag client traffic with a client
VLAN and AP traffic with a management VLAN.  This was achieved enabling
"Use VLAN Tagging" on the `DEFAULT_RLAN` with the management VLAN ID.  The
`default-group` Access Points Group assigned each physical network port to
`DEFAULT_RLAN`.  AP clients were then assigned to a client VLAN by enabling
"Use VLAN Tagging" on each WLAN with the VLAN ID set to the client VLAN.

If I download my config the RLAN looks like this:

```
config remote-lan interface 16 management
config remote-lan flexconnect local-switching 16 vlan 231
config remote-lan flexconnect local-switching 16 enable
config remote-lan create 16 DEFAULT_RLAN DEFAULT_RLAN
config remote-lan enable 16
...
config flexconnect group default-flexgroup wlan-vlan wlan 16 add vlan 231
```

And the resulting VLAN template and WiFi networks look like this:

```
config flexconnect vlan-name-id template-entry add vlan200_200 vlan200 200
config flexconnect vlan-name-id create vlan200_200
config flexconnect vlan-name-id apply vlan200_200
...
config flexconnect group default-flexgroup wlan-vlan wlan 2 add vlan 200
config flexconnect group default-flexgroup wlan-vlan wlan 3 add vlan 200
config flexconnect group default-flexgroup wlan-vlan wlan 4 add vlan 200
```

### Configuring WiFi VLANs

If you specify a VLAN in a Radius response, and the AP isn't already aware of that
VLAN, client traffic will be assigned to the default VLAN for the WLAN.  This is hard
to debug since the UI shows that a VLAN has been assigned to the client.

There is no interface to directly manage VLANs on the APs, but there are several ways
to create them.  As you can see above, assigning a VLAN to a WLAN creates the VLAN.
You can also create them by adding RLANs:

*  In "Wireless Settings" select "WLANS" and click "Add new WLAN/RLAN"
*  In the "General tab select Type `RLAN` and set the Profile name to the VLAN number.  The Network ID field doesn't matter.
*  In the "VLAN and Firewall" tab set "Use VLAN Tagging" to `Yes` and set "VLAN ID" to the VLAN number.
*  Press "Apply"

This is what the resulting config looks like for VLAN 202:

```
config remote-lan avc 12 visibility enable
config remote-lan radius_server auth disable 12
config remote-lan interface 12 management
config remote-lan flexconnect local-switching 12 vlan 202
config remote-lan flexconnect local-switching 12 enable
config remote-lan security web-auth server-precedence 12 local radius ldap
config remote-lan session-timeout 12 0
config remote-lan create 12 202 202
config remote-lan exclusionlist 12 180
config remote-lan enable 12
...
config flexconnect vlan-name-id template-entry add vlan202_202 vlan202 202
config flexconnect vlan-name-id create vlan202_202
config flexconnect vlan-name-id apply vlan202_202
...
config flexconnect group default-flexgroup wlan-vlan wlan 12 add vlan 202
```

If you have split your APs into multiple Access Point Groups, you will need to go
into the AP Groups settings and add the new RLANs to each AP Group that will host
identity networking.  If you don't do this, the APs in custom groups won't know about
the VLANs.  The default AP group gets every new RLAN by default.

### Enable Radius and AAA Override on WLANs

To use Radius you need to switch your WLANs to "WPA2 Enterprise".  This enables you
to set "Authentication Server" to "External Radius" and add Radius servers.

To accept and use VLANs from Radius, you need to be in expert mode.  In the "Advanced"
tab of the WLAN configuration set "Allow AAA Override" to enabled.  This allows the
Radius server to specify the VLAN for each client.

In the "VLAN and Firewall" tab of your WLANs you can set "Use VLAN Tagging" to disabled
or enabled, this does not affect AAA Override.  You should enable it though and set the
VLAN ID to the client VLAN.  If you leave it disabled then clients without a Radius
assigned VLAN will end up on the management VLAN (provided by the default RLAN).

Here's the resulting config:

```
config wlan aaa-override enable 2
config wlan radius_server acct add 2 1
config wlan radius_server auth add 2 1
config wlan radius_server auth caching enable 2
```

## FreeRadius Overview

Here are the major changes that are needed to the FreeRadius 3.0 out of box config.

https://github.com/eiddor/cisco-sda-freeradius looks like a good starting point, but
if you use EAP be sure to set `use_tunneled_reply`.

### EAP

Activate `/etc/freeradius/3.0/mods-available/eap` and make these changes:

*  Uncomment the EAP-pwd section to enable password auth.
*  Point private_key_file etc at your TLS certificates.
*  Set `auto_chain` to yes so that the server provides the certificate chain.
*  Set `check_crl` to no if you are using a non public CA.
*  Set `disable_tlsv1` to yes and `tls_min_version` to 1.1 for security.
*  Set `use_tunneled_reply` to yes in two places.

It's really important to set `use_tunneled_reply` to yes, otherwise the APs
will get their Radius attributes from the outside (anonymous) tunnel rather than
the inside (user) tunnel.  If you get this wrong it looks like the APs ignore
your attributes, when in reality they never see them.

#### Certs

A quick note on certs.  `/etc/freeradius/3.0/certs` contains scripts to generate
CA, server and client certs.  I found it useful to modify the configs so that
"xpextensions" included the authority key ID in the server cert so that Android
clients could verify the chain:
`authorityKeyIdentifier = keyid:always,issuer:always`

If you use those configs you will want to go through them first to set the names
and validity periods for everything.

### Clients

`/etc/freeradius/3.0/clients.conf` needs to be configured with the AP floating
IP address:

```
client cbw {
      ipaddr          = 192.168.0.253
      secret          = "..."
}
```

### Users

Radius responses need to include Tunnel-Type VLAN (13) and Tunnel-Medium-Type
IEEE-802 (13).  Here's what a typical user looks like in `/etc/freeradius/3.0/users`:

```
username Cleartext-Password := "...
         Service-Type = Framed,
         Tunnel-Type:0 = 13,
         Tunnel-Medium-Type:0 = 6,
         Tunnel-Private-Group-Id:0 = 202
```

#### A note on VLAN names

From the AP config you might expect that `vlan202_202` or `vlan202` would be valid VLAN
names to supply with Radius, but using them results traffic going to the default VLAN and
in this log message from the APs: `May 10 13:36:42.950: [ERROR] apf_policy.c 5210: Either Vlan Name id Template invalid or   no name to id mapping exist for interface 'vlan201_201'`

Using numbers for VLAN names also results in a similar log message, but the VLAN assignment
works: `May 10 13:41:14.967: [ERROR] apf_policy.c 5210: Either Vlan Name id Template invalid or   no name to id mapping exist for interface '201'`

### Setting QoS, etc

If you append [dictionary.airespace](https://github.com/retailnext/node-radius/blob/master/test/dictionaries/dictionary.airespace)
to `/etc/freeradius/3.0/dictionary` then you can provide other attributes such as
ACL names and QoS values with Radius.

