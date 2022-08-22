---
title: "My Basic MikroTik hEX S Setup"
date: 2022-08-13T21:53:04+02:00
draft: false
tags: ["mikrotik", "homelab", "vlans"]
summary: How I configured a MikroTik hEX S router with VLANs to separate different parts of my home network.
---
Once I began working on my homelab, I realised that my trivial network setup
wouldn't meet my needs anymore and it needs to be improved.

Below I'll describe how I segmented the network to restrict access to different
resources for my home users using a MikroTik hEX S router. 

## Objectives
The home network can be viewed from two angles. One is their users and their
needs. Apart from the fully trusted family members, there are guests that
simply need the internet access over WiFi and Work From Home users, that would
like to reach the internet, but could also use selected shared resources,
e.g. a printer.

The second point of view could from the perspective of the resources that
should be accessible to their users. Here we have the aforementioned printer,
but also file sharing services, git hosting and possibly others. Some of them
might contain sensitive data and thus access to them should be restricted.

In my case, different user groups and shared network resource types will be
placed in separate VLANs and the router will be configured to allow only the
necessary traffic between them. There is no need for the user groups to talk
with eachother, so that will be blocked.

The end result should look more or less like this:

```goat
                                Internet
                                    |
    +--------+                      |
    | Guests |----------------------+
    +--------+                      |
                                    |
    +-------------------+           |
    | Work(ers)FromHome |-----------+
    +-------------------+           |
              |                     |
              +---------------------|---------- Printer                          
                                    |              |
    +--------------------+          |              |
    | Trusted Home Users |----------+              |
    +--------------------+                         |
          |   |                                    |
          |   +------------------------------------+
          +---------------------------------------------------- HomeLab
```

The configuration will be done in the terminal. This will make it easier for me
to understand the config export and version it properly in git.

## Default configuration (or where we start from)
The first time setup is described in detail in the [MikroTik
wiki](https://wiki.mikrotik.com/wiki/Manual:First_time_startup).

Basically, for the hEX S, the `ether2`-`ether5` ports are connected into a bridge (named `bridge`).
The DHCP server is enabled and gives out IP addresses in the `192.168.88.0/24`
network, with the `192.168.88.1` being the address of the hEX S itself.

The Web UI is available under [http://192.168.88.1](http://192.168.88.1). The
default username is `admin` with no password. You can also ssh into the box
with `ssh 192.168.88.1 -l admin`

Running `/export` in the routers terminal will pring out the current configuration.

{{< details "Click to see the default config of my hEX S" >}}
```
# aug/15/2022 xx:xx:xx by RouterOS 7.4.1
# software id = BLFX-WCHT
#
# model = RB760iGS
# serial number = XXXXXXXXXXXX
/interface bridge
add admin-mac=xx:xx:xx:xx:xx:xx auto-mac=no comment=defconf name=bridge
/interface list
add comment=defconf name=WAN
add comment=defconf name=LAN
/interface wireless security-profiles
set [ find default=yes ] supplicant-identity=MikroTik
/ip hotspot profile
set [ find default=yes ] html-directory=hotspot
/ip pool
add name=default-dhcp ranges=192.168.88.10-192.168.88.254
/ip dhcp-server
add address-pool=default-dhcp interface=bridge name=defconf
/port
set 0 name=serial0
/interface bridge port
add bridge=bridge comment=defconf interface=ether2
add bridge=bridge comment=defconf interface=ether3
add bridge=bridge comment=defconf interface=ether4
add bridge=bridge comment=defconf interface=ether5
add bridge=bridge comment=defconf interface=sfp1
/ip neighbor discovery-settings
set discover-interface-list=LAN
/interface list member
add comment=defconf interface=bridge list=LAN
add comment=defconf interface=ether1 list=WAN
/ip address
add address=192.168.88.1/24 comment=defconf interface=bridge network=192.168.88.0
/ip dhcp-client
add comment=defconf interface=ether1
/ip dhcp-server network
add address=192.168.88.0/24 comment=defconf dns-server=192.168.88.1 gateway=192.168.88.1
/ip dns
set allow-remote-requests=yes
/ip dns static
add address=192.168.88.1 comment=defconf name=router.lan
/ip firewall filter
add action=accept chain=input comment="defconf: accept established,related,untracked" connection-state=established,related,untracked
add action=drop chain=input comment="defconf: drop invalid" connection-state=invalid
add action=accept chain=input comment="defconf: accept ICMP" protocol=icmp
add action=accept chain=input comment="defconf: accept to local loopback (for CAPsMAN)" dst-address=127.0.0.1
add action=drop chain=input comment="defconf: drop all not coming from LAN" in-interface-list=!LAN
add action=accept chain=forward comment="defconf: accept in ipsec policy" ipsec-policy=in,ipsec
add action=accept chain=forward comment="defconf: accept out ipsec policy" ipsec-policy=out,ipsec
add action=fasttrack-connection chain=forward comment="defconf: fasttrack" connection-state=established,related hw-offload=yes
add action=accept chain=forward comment="defconf: accept established,related, untracked" connection-state=established,related,untracked
add action=drop chain=forward comment="defconf: drop invalid" connection-state=invalid
add action=drop chain=forward comment="defconf: drop all from WAN not DSTNATed" connection-nat-state=!dstnat connection-state=new in-interface-list=WAN
/ip firewall nat
add action=masquerade chain=srcnat comment="defconf: masquerade" ipsec-policy=out,none out-interface-list=WAN
/ipv6 firewall address-list
add address=::/128 comment="defconf: unspecified address" list=bad_ipv6
add address=::1/128 comment="defconf: lo" list=bad_ipv6
add address=fec0::/10 comment="defconf: site-local" list=bad_ipv6
add address=::ffff:0.0.0.0/96 comment="defconf: ipv4-mapped" list=bad_ipv6
add address=::/96 comment="defconf: ipv4 compat" list=bad_ipv6
add address=100::/64 comment="defconf: discard only " list=bad_ipv6
add address=2001:db8::/32 comment="defconf: documentation" list=bad_ipv6
add address=2001:10::/28 comment="defconf: ORCHID" list=bad_ipv6
add address=3ffe::/16 comment="defconf: 6bone" list=bad_ipv6
/ipv6 firewall filter
add action=accept chain=input comment="defconf: accept established,related,untracked" connection-state=established,related,untracked
add action=drop chain=input comment="defconf: drop invalid" connection-state=invalid
add action=accept chain=input comment="defconf: accept ICMPv6" protocol=icmpv6
add action=accept chain=input comment="defconf: accept UDP traceroute" port=33434-33534 protocol=udp
add action=accept chain=input comment="defconf: accept DHCPv6-Client prefix delegation." dst-port=546 protocol=udp src-address=fe80::/10
add action=accept chain=input comment="defconf: accept IKE" dst-port=500,4500 protocol=udp
add action=accept chain=input comment="defconf: accept ipsec AH" protocol=ipsec-ah
add action=accept chain=input comment="defconf: accept ipsec ESP" protocol=ipsec-esp
add action=accept chain=input comment="defconf: accept all that matches ipsec policy" ipsec-policy=in,ipsec
add action=drop chain=input comment="defconf: drop everything else not coming from LAN" in-interface-list=!LAN
add action=accept chain=forward comment="defconf: accept established,related,untracked" connection-state=established,related,untracked
add action=drop chain=forward comment="defconf: drop invalid" connection-state=invalid
add action=drop chain=forward comment="defconf: drop packets with bad src ipv6" src-address-list=bad_ipv6
add action=drop chain=forward comment="defconf: drop packets with bad dst ipv6" dst-address-list=bad_ipv6
add action=drop chain=forward comment="defconf: rfc4890 drop hop-limit=1" hop-limit=equal:1 protocol=icmpv6
add action=accept chain=forward comment="defconf: accept ICMPv6" protocol=icmpv6
add action=accept chain=forward comment="defconf: accept HIP" protocol=139
add action=accept chain=forward comment="defconf: accept IKE" dst-port=500,4500 protocol=udp
add action=accept chain=forward comment="defconf: accept ipsec AH" protocol=ipsec-ah
add action=accept chain=forward comment="defconf: accept ipsec ESP" protocol=ipsec-esp
add action=accept chain=forward comment="defconf: accept all that matches ipsec policy" ipsec-policy=in,ipsec
add action=drop chain=forward comment="defconf: drop everything else not coming from LAN" in-interface-list=!LAN
/system clock
set time-zone-name=Europe/Warsaw
/system routerboard settings
set silent-boot=yes
/tool mac-server
set allowed-interface-list=LAN
/tool mac-server mac-winbox
set allowed-interface-list=LAN
```
{{< /details >}}

> **Note:** It might be a good idea to put the config export into a git repository and keep it up to date there.

I'll be starting off with this default setup and modifying it to meet my needs.
With that approach, I start with something that already works and apply changes that are easy for me to understand. 

## Hardening the router
The default setup is insecure. Please follow the [hardening
guide](https://wiki.mikrotik.com/wiki/Manual:Securing_Your_Router) or at the very least change the `admin` user password with:
```
/user set [find name=admin] password=CorrectHorseBatteryStaple
```

Check out [this xkcd comic](https://xkcd.com/936/) if you aren't familiar with the `correct horse battery staple`.

## In case something goes wrong...
In order to go back to the initial configuration, you can either run `/system
reset-configuration` in the router terminal or follow this
[guide](https://wiki.mikrotik.com/wiki/Manual:Reset).

## Management port setup
I'll begin by configuring a dedicated management port to avoid being
disconnected from the router during the configuration and to make it easy for
me to access it later if something goes wrong.

In my case, the management port will be `ether2`. Make sure you are connected to one of the other LAN ports and then remove `ether2` from the bridge:
```
/interface/bridge/port/remove [find interface=ether2]
```

Add an IP address to that interface:
```
/ip/address/add interface=ether2 address=192.168.0.1 netmask=255.255.255.0
```
and create an IP address pool to be given out by the DHCP server:
```
/ip/pool/add name=mgmt-dhcp ranges=192.168.0.2-192.168.0.254
```
as well as the DHCP network:
```
/ip/dhcp-server/network/add address=192.168.0.0/24 comment=mgmt
```

followed by configuring the DHCP server itself:
```
/ip/dhcp-server/add interface=ether2 address-pool=mgmt-dhcp name=mgmt-dhcp
```

Is that all? No. Let's take a look at the firewall rules:
```
[admin@MikroTik] > /ip/firewall/filter/print chain=input
Flags: X - disabled, I - invalid; D - dynamic 
 0    ;;; defconf: accept established,related,untracked
      chain=input action=accept connection-state=established,related,untracked 

 1    ;;; defconf: drop invalid
      chain=input action=drop connection-state=invalid 

 2    ;;; defconf: accept ICMP
      chain=input action=accept protocol=icmp 

 3    ;;; defconf: accept to local loopback (for CAPsMAN)
      chain=input action=accept dst-address=127.0.0.1 

 4    ;;; defconf: drop all not coming from LAN
      chain=input action=drop in-interface-list=!LAN 
[admin@MikroTik] > 
```
The LAN interface list contains the `bridge` interface:
```
[admin@MikroTik] > /interface/list/member/print 
Columns: LIST, INTERFACE
# LIST  INTERFACE
;;; defconf
0 LAN   bridge   
;;; defconf
1 WAN   ether1   
[admin@MikroTik] >
```
As I removed the `ether2` from the `bridge`, the incoming traffic from that interface will fall into the 4th firewall rule and be dropped.

In order to fix that, add a new rule that explicitly allows access from the `ether2`:
```
/ip/firewall/filter/add chain=input action=accept in-interface=ether2 place-before=4
```
> **Note:** In MikroTik CLI temporary numeric item identifiers are assigned after the `print` command and are valid until the next `print`. Here the `4` is used because that's what was returned in the print above.

It should be safe now to connect to the `ether2` port and continue the configuration from there. Please note that it uses a different IP address, so you need to connect to `192.168.0.1` instead of the default IP address that was used before.

## Default config cleanup
First, let's remove things from the default config that we won't need anymore.

I'm connected through the dedicated management port now, so I can setup the
bridge without worrying about losing the connection with the router.

DHCP server on the `bridge` interface? Not needed anymore, there'll be a
separate DHCP server for each VLAN. Remove it and the related resources:
```
/ip/dhcp-server/remove [find name=defconf]
/ip/pool/remove [find name=default-dhcp]
/ip/dhcp-server/network/remove [find comment=defconf]
```

The IP address of the `bridge` and the DNS entry pointing to it are also to be removed:
```
/ip/address/remove [find comment=defconf]
/ip/dns/static/remove [find comment=defconf]
```

## VLANs
As I understood from the [L2
misconfiguration](https://help.mikrotik.com/docs/display/ROS/Layer2+misconfiguration)
and the [Bridge VLAN
Table](https://help.mikrotik.com/docs/display/ROS/Bridge+VLAN+Table) pages of
the RouterOS documentation, the [Bridge VLAN
Filtering](https://help.mikrotik.com/docs/display/ROS/Bridging+and+Switching#BridgingandSwitching-BridgeVLANFiltering)
is the way to go in order to setup the VLANs and the inter-VLAN routing.

With RouterOS v7 this is also [hardware accelerated on hEX
S](https://help.mikrotik.com/docs/display/ROS/Bridging+and+Switching#BridgingandSwitching-BridgeVLANFiltering),
so make sure you upgrade the OS.

Let's begin by assigning VLAN IDs for the groups mentioned above:
```goat
                                  +----------+---------+                                                    
                                  |Name      | VLAN ID |                                                    
                                  +==========+=========+                                                    
                                  |Guests    |    10   |                                                    
                                  +----------+---------+                                                    
                                  |WFH       |    20   |                                                    
                                  +----------+---------+                                                    
                                  |HomeUsers |    30   |                                                    
                                  +----------+---------+                                                    
                                  |Shared    |    40   |                                                    
                                  +----------+---------+                                                    
                                  |HomeLab   |    50   |                                                    
                                  +----------+---------+                                                    
                                  |Management|    60   |                                                    
                                  +----------+---------+                                                    
```
Please note the addition of the `Management` VLAN that will be used to access
the management interfaces of the switches.

In my case, the physical connections on the hEX S will look like this:

```goat
     
     +-- hEX S -------------------------+                                    
     |                                  |                                    
     |   SFP      1   2   3   4   5     |                                    
     |  +---+   +---+---+---+---+---+   |                                    
     |  | x |   | o | o | o | o | o |   |                                    
     |  +---+   +-+-+-+-+-+-+-+-+-+-+   |                                    
     |            |   |   |   |   |     |                                    
     +----------------------------------+                                    
                  |   |   |   |   |                                          
          WAN  ---+   |   |   |   +-- Access Point     (Tagged VLANs: 10, 20, 30)                          
                      |   |   |                                              
                      |   |   +------ 'Main' Switch    (Tagged VLANs: 10, 20, 30, 40, 50, 60)                         
                      |   |                                                  
                      |   +---------- HomeLab Switch   (Untagged VLANs: 50)                        
                      |                                                      
                      +-------------- Management Port                        
```

After configuring the `ether2` as the management port, the `bridge` spans ports `ether3` to `ether5`.

By default the `vlan-filtering` on the `bridge` interface is off, switch it on:
```
/interface/bridge/set bridge vlan-filtering=yes
```

I'd like to have a DHCP server running for all the VLANs, so I'll create an
interface for each:
```
/interface/vlan/add interface=bridge name=vlan10 vlan-id=10
/interface/vlan/add interface=bridge name=vlan20 vlan-id=20
/interface/vlan/add interface=bridge name=vlan30 vlan-id=30
/interface/vlan/add interface=bridge name=vlan40 vlan-id=40
/interface/vlan/add interface=bridge name=vlan50 vlan-id=50
/interface/vlan/add interface=bridge name=vlan60 vlan-id=60
```
and assign IP addresses (I'd like to have a gateway in each VLAN):
```
/ip/address/add address=192.168.10.1/24 interface=vlan10
/ip/address/add address=192.168.20.1/24 interface=vlan20
/ip/address/add address=192.168.30.1/24 interface=vlan30
/ip/address/add address=192.168.40.1/24 interface=vlan40
/ip/address/add address=192.168.50.1/24 interface=vlan50
/ip/address/add address=192.168.60.1/24 interface=vlan60
```

Let's focus on the `HomeLab` network for a moment. It will be accessible
without tagging on `ether3` and should be the easiest to test.

First, configure the `pvid` on the bridge port, to assign the (untagged)
traffic from that port to the correct VLAN:
```
/interface/bridge/port/set pvid=50 numbers=[find where interface=ether3]
```
Now add the VLAN to the bridge. Please note, that I'm adding the `bridge` as a
tagged interface here. This is required to connect the DHCP server for that
VLAN to the network:
```
/interface/bridge/vlan/add bridge=bridge tagged=bridge untagged=ether3 vlan-ids=50
```
Wait... Did I say a DHCP server?
```shell
# it needs a pool of addresses to give to the clients:
/ip/pool/add name=vlan50-dhcp ranges=192.168.50.2-192.168.50.254
# it would be nice to setup the default gateway well:
/ip/dhcp-server/network/add address=192.168.50.0/24 gateway=192.168.50.1
# and the server itself:
/ip/dhcp-server/add address-pool=vlan50-dhcp interface=vlan50 name=dhcp-vlan50
```
The VLAN 50 should be functional now, with DHCP handing out addresses to the
clients and being able to access the internet through the gateway.

Give it a try, there is no point in going any further if that doesn't work.

...

Ok, if that is working as intended, let's jump to the tagged VLANs setup.

As before, add the VLANs to the bridge:

```csh
# I'll have wired and wireless users:
/interface/bridge/vlan/add bridge=bridge vlan-ids=10 tagged=ether4,ether5,bridge
/interface/bridge/vlan/add bridge=bridge vlan-ids=20 tagged=ether4,ether5,bridge
/interface/bridge/vlan/add bridge=bridge vlan-ids=30 tagged=ether4,ether5,bridge
# but only wired 'services':
/interface/bridge/vlan/add bridge=bridge vlan-ids=40 tagged=ether4,bridge
/interface/bridge/vlan/add bridge=bridge vlan-ids=60 tagged=ether4,bridge
```
and configure the DHCP:
```
/ip/pool/add name=vlan10-dhcp ranges=192.168.10.2-192.168.10.254
/ip/pool/add name=vlan20-dhcp ranges=192.168.20.2-192.168.20.254
/ip/pool/add name=vlan30-dhcp ranges=192.168.30.2-192.168.30.254
/ip/pool/add name=vlan40-dhcp ranges=192.168.40.2-192.168.40.254
/ip/pool/add name=vlan60-dhcp ranges=192.168.60.2-192.168.60.254

/ip/dhcp-server/network/add address=192.168.10.0/24 gateway=192.168.10.1 dns-server=192.168.10.1
/ip/dhcp-server/network/add address=192.168.20.0/24 gateway=192.168.20.1 dns-server=192.168.20.1
/ip/dhcp-server/network/add address=192.168.30.0/24 gateway=192.168.30.1 dns-server=192.168.30.1
/ip/dhcp-server/network/add address=192.168.40.0/24 gateway=192.168.40.1 dns-server=192.168.40.1
/ip/dhcp-server/network/add address=192.168.60.0/24 gateway=192.168.60.1 dns-server=192.168.60.1

/ip/dhcp-server/add address-pool=vlan10-dhcp interface=vlan10 name=dhcp-vlan10
/ip/dhcp-server/add address-pool=vlan20-dhcp interface=vlan20 name=dhcp-vlan20
/ip/dhcp-server/add address-pool=vlan30-dhcp interface=vlan30 name=dhcp-vlan30
/ip/dhcp-server/add address-pool=vlan40-dhcp interface=vlan40 name=dhcp-vlan40
/ip/dhcp-server/add address-pool=vlan60-dhcp interface=vlan60 name=dhcp-vlan60
```

Now, let's make sure that only the traffic we expect is coming from the ports:
```shell
# untagged home lab:
/interface/bridge/port/set [find where interface=ether3] ingress-filtering=yes \
  frame-types=admit-only-untagged-and-priority-tagged
# tagged switch and ap:
/interface/bridge/port/set [find where interface=ether4] ingress-filtering=yes \
  frame-types=admit-only-vlan-tagged
/interface/bridge/port/set [find where interface=ether5] ingress-filtering=yes \
  frame-types=admit-only-vlan-tagged
```
The
[documentation](https://help.mikrotik.com/docs/display/ROS/Bridge+VLAN+Table)
says it's a good idea to allow only tagged traffic to the bridge interface, so
let's do that:
```
/interface/bridge/set bridge frame-types=admit-only-vlan-tagged ingress-filtering=yes
```
I don't think it's strictly necessary in this setup though. The `bridge`
doesn't have an IP assigned and the router is connected to the other networks
anyway, but I'm adding the filter anyway in case I miss something obvious.

The VLANs are connected to the router, they should be able to reach the
internet... Are we done? Not yet! I wanted to isolate the hosts connected to
the VLANs, but at this point they are able to talk to each other through the
router. Why?

## Filtering the traffic
There are no firewall rules that would restrict the traffic flowing through the
router between the VLANs. That has to be fixed.

First, let's remove the IPv6 from the scope. I don't need it for now:
```shell
# remove all ipv6 firewall rules
/ipv6/firewall/filter/remove [find]
# drop all traffic:
/ipv6/firewall/filter/add chain=forward action=drop
/ipv6/firewall/filter/add chain=input action=drop
```
Done. Problem solved.

Now, how does the default firewall for IPv4 look like?
```shell
[admin@MikroTik] /ip/pool> /ip/firewall/filter/print chain=forward
Flags: X - disabled, I - invalid; D - dynamic 
 0  D ;;; special dummy rule to show fasttrack counters
      chain=forward action=passthrough 

 1    ;;; defconf: accept in ipsec policy
      chain=forward action=accept ipsec-policy=in,ipsec 

 2    ;;; defconf: accept out ipsec policy
      chain=forward action=accept ipsec-policy=out,ipsec 

 3    ;;; defconf: fasttrack
      chain=forward action=fasttrack-connection hw-offload=yes connection-state=established,related 

 4    ;;; defconf: accept established,related, untracked
      chain=forward action=accept connection-state=established,related,untracked 

 5    ;;; defconf: drop invalid
      chain=forward action=drop connection-state=invalid 

 6    ;;; defconf: drop all from WAN not DSTNATed
      chain=forward action=drop connection-state=new connection-nat-state=!dstnat in-interface-list=WAN 
```

Let's add a drop catch-all rule at the end:
```
/ip/firewall/filter/add chain=forward action=drop comment=catchall
```
I assume there is no internet access from the VLANs now. Grant it with:
```
/ip/firewall/filter/add chain=forward action=accept connection-state=new out-interface-list=WAN in-interface-list=!WAN place-before=[find comment=catchall]
```
Now allow the connections between VLANs, as shown in the diagram at the beginning of this post:
```
/ip/firewall/filter/add chain=forward action=accept connection-state=new out-interface=vlan40 in-interface=vlan20 place-before=[find comment=catchall]
/ip/firewall/filter/add chain=forward action=accept connection-state=new out-interface=vlan40 in-interface=vlan30 place-before=[find comment=catchall]
/ip/firewall/filter/add chain=forward action=accept connection-state=new out-interface=vlan50 in-interface=vlan30 place-before=[find comment=catchall]
```

I'm fine with home users being able to access the management VLAN as well:
```
/ip/firewall/filter/add chain=forward action=accept connection-state=new out-interface=vlan60 in-interface=vlan30 place-before=[find comment=catchall]
```

## DNS Haiku
There is still something missing here...

> It's not DNS
>
> There's no way it's DNS
>
> It was DNS
>
>[*source*](https://www.cyberciti.biz/humour/a-haiku-about-dns/)

The DNS server is enabled by default, but after reconfiguring the bridge, the input firewall rules need some adjustments. Let's take a look:
```
 4    ;;; defconf: drop all not coming from LAN
      chain=input action=drop in-interface-list=!LAN 
```

The `LAN` interface list contains the bridge interface, but it would make more
sense to have the VLAN interfaces there. Do I need all of the traffic though?
DNS should be enough for now. Instead of creating single rule for each VLAN, I'll create an interface list:
```
/interface/list/add name=all-internal-vlans
/interface/list/member/add list=all-internal-vlans interface=vlan10
/interface/list/member/add list=all-internal-vlans interface=vlan20
/interface/list/member/add list=all-internal-vlans interface=vlan30
/interface/list/member/add list=all-internal-vlans interface=vlan40
/interface/list/member/add list=all-internal-vlans interface=vlan50
/interface/list/member/add list=all-internal-vlans interface=vlan60
```
and then add the firewall rule:
```
/ip/firewall/filter/add chain=input action=accept dst-port=53 protocol=udp in-interface-list=all-internal-vlans place-before=[find comment=catchall]
```

I'll finish by removing the default rule mentioned above:
```
/ip/firewall/filter/remove [find comment="defconf: drop all not coming from LAN"]
```
and replacing it with a drop all rule:
```
/ip/firewall/filter add chain=input action=drop
```

## Summary

{{< details "Click here to see the resulting config" >}}

```
# aug/16/2022 13:37:11 by RouterOS 7.4.1
# software id = BLFX-WCHT
#
# model = RB760iGS
# serial number = XXXXXXXXXXXX
/interface bridge
add admin-mac=xx:xx:xx:xx:xx:xx auto-mac=no comment=defconf frame-types=admit-only-vlan-tagged name=bridge vlan-filtering=yes
/interface vlan
add interface=bridge name=vlan10 vlan-id=10
add interface=bridge name=vlan20 vlan-id=20
add interface=bridge name=vlan30 vlan-id=30
add interface=bridge name=vlan40 vlan-id=40
add interface=bridge name=vlan50 vlan-id=50
add interface=bridge name=vlan60 vlan-id=60
/interface list
add comment=defconf name=WAN
add comment=defconf name=LAN
add name=all-internal-vlans
/interface wireless security-profiles
set [ find default=yes ] supplicant-identity=MikroTik
/ip hotspot profile
set [ find default=yes ] html-directory=hotspot
/ip pool
add name=mgmt-dhcp ranges=192.168.0.2-192.168.0.254
add name=vlan50-dhcp ranges=192.168.50.2-192.168.50.254
add name=vlan10-dhcp ranges=192.168.10.2-192.168.10.254
add name=vlan20-dhcp ranges=192.168.20.2-192.168.20.254
add name=vlan30-dhcp ranges=192.168.30.2-192.168.30.254
add name=vlan40-dhcp ranges=192.168.40.2-192.168.40.254
add name=vlan60-dhcp ranges=192.168.60.2-192.168.60.254
/ip dhcp-server
add address-pool=mgmt-dhcp interface=ether2 name=mgmt-dhcp
add address-pool=vlan50-dhcp interface=vlan50 name=dhcp-vlan50
add address-pool=vlan10-dhcp interface=vlan10 name=dhcp-vlan10
add address-pool=vlan20-dhcp interface=vlan20 name=dhcp-vlan20
add address-pool=vlan30-dhcp interface=vlan30 name=dhcp-vlan30
add address-pool=vlan40-dhcp interface=vlan40 name=dhcp-vlan40
add address-pool=vlan60-dhcp interface=vlan60 name=dhcp-vlan60
/port
set 0 name=serial0
/interface bridge port
add bridge=bridge comment=defconf frame-types=admit-only-untagged-and-priority-tagged interface=ether3 pvid=50
add bridge=bridge comment=defconf frame-types=admit-only-vlan-tagged interface=ether4
add bridge=bridge comment=defconf frame-types=admit-only-vlan-tagged interface=ether5
add bridge=bridge comment=defconf interface=sfp1
/ip neighbor discovery-settings
set discover-interface-list=LAN
/interface bridge vlan
add bridge=bridge tagged=bridge untagged=ether3 vlan-ids=50
add bridge=bridge tagged=ether4,ether5,bridge vlan-ids=10
add bridge=bridge tagged=ether4,ether5,bridge vlan-ids=20
add bridge=bridge tagged=ether4,ether5,bridge vlan-ids=30
add bridge=bridge tagged=ether4,bridge vlan-ids=40
add bridge=bridge tagged=ether4,bridge vlan-ids=60
/interface list member
add comment=defconf interface=bridge list=LAN
add comment=defconf interface=ether1 list=WAN
add interface=vlan10 list=all-internal-vlans
add interface=vlan20 list=all-internal-vlans
add interface=vlan30 list=all-internal-vlans
add interface=vlan40 list=all-internal-vlans
add interface=vlan50 list=all-internal-vlans
add interface=vlan60 list=all-internal-vlans
/ip address
add address=192.168.0.1/24 interface=ether2 network=192.168.0.0
add address=192.168.10.1/24 interface=vlan10 network=192.168.10.0
add address=192.168.20.1/24 interface=vlan20 network=192.168.20.0
add address=192.168.30.1/24 interface=vlan30 network=192.168.30.0
add address=192.168.40.1/24 interface=vlan40 network=192.168.40.0
add address=192.168.50.1/24 interface=vlan50 network=192.168.50.0
add address=192.168.60.1/24 interface=vlan60 network=192.168.60.0
/ip dhcp-client
add comment=defconf interface=ether1
/ip dhcp-server network
add address=192.168.0.0/24 comment=mgmt
add address=192.168.10.0/24 dns-server=192.168.10.1 gateway=192.168.10.1
add address=192.168.20.0/24 dns-server=192.168.20.1 gateway=192.168.20.1
add address=192.168.30.0/24 dns-server=192.168.30.1 gateway=192.168.30.1
add address=192.168.40.0/24 dns-server=192.168.40.1 gateway=192.168.40.1
add address=192.168.50.0/24 gateway=192.168.50.1
add address=192.168.60.0/24 dns-server=192.168.60.1 gateway=192.168.60.1
/ip dns
set allow-remote-requests=yes
/ip firewall filter
add action=accept chain=input comment="defconf: accept established,related,untracked" connection-state=established,related,untracked
add action=drop chain=input comment="defconf: drop invalid" connection-state=invalid
add action=accept chain=input comment="defconf: accept ICMP" protocol=icmp
add action=accept chain=input in-interface=ether2
add action=accept chain=input comment="defconf: accept to local loopback (for CAPsMAN)" dst-address=127.0.0.1
add action=accept chain=forward comment="defconf: accept in ipsec policy" ipsec-policy=in,ipsec
add action=accept chain=forward comment="defconf: accept out ipsec policy" ipsec-policy=out,ipsec
add action=fasttrack-connection chain=forward comment="defconf: fasttrack" connection-state=established,related hw-offload=yes
add action=accept chain=forward comment="defconf: accept established,related, untracked" connection-state=established,related,untracked
add action=drop chain=forward comment="defconf: drop invalid" connection-state=invalid
add action=drop chain=forward comment="defconf: drop all from WAN not DSTNATed" connection-nat-state=!dstnat connection-state=new in-interface-list=WAN
add action=accept chain=forward connection-state=new in-interface-list=!WAN out-interface-list=WAN
add action=accept chain=forward connection-state=new in-interface=vlan20 out-interface=vlan40
add action=accept chain=forward connection-state=new in-interface=vlan30 out-interface=vlan40
add action=accept chain=forward connection-state=new in-interface=vlan30 out-interface=vlan50
add action=accept chain=forward connection-state=new in-interface=vlan30 out-interface=vlan60
add action=accept chain=input dst-port=53 in-interface-list=all-internal-vlans protocol=udp
add action=drop chain=forward comment=catchall
add action=drop chain=input
/ip firewall nat
add action=masquerade chain=srcnat comment="defconf: masquerade" ipsec-policy=out,none out-interface-list=WAN
/ipv6 firewall address-list
add address=::/128 comment="defconf: unspecified address" list=bad_ipv6
add address=::1/128 comment="defconf: lo" list=bad_ipv6
add address=fec0::/10 comment="defconf: site-local" list=bad_ipv6
add address=::ffff:0.0.0.0/96 comment="defconf: ipv4-mapped" list=bad_ipv6
add address=::/96 comment="defconf: ipv4 compat" list=bad_ipv6
add address=100::/64 comment="defconf: discard only " list=bad_ipv6
add address=2001:db8::/32 comment="defconf: documentation" list=bad_ipv6
add address=2001:10::/28 comment="defconf: ORCHID" list=bad_ipv6
add address=3ffe::/16 comment="defconf: 6bone" list=bad_ipv6
/ipv6 firewall filter
add action=drop chain=forward
add action=drop chain=input
/system clock
set time-zone-name=Europe/Warsaw
/system routerboard settings
set silent-boot=yes
/tool mac-server
set allowed-interface-list=LAN
/tool mac-server mac-winbox
set allowed-interface-list=LAN
```
{{< /details >}}

That's it!

The end result works as intended and I'm pleased with the outcome. It's a good
cornerstone for my homelab.

This relatively simple task allowed me to get familiar with the MikroTik CLI.
I really like the fact that I can go through the full config export and make
sure that it doesn't contain any surprises.

Thank you for reading and have a great day!

