---
title: EVPN Multihoming
author: NVIDIA
weight: 570
toc: 4
---

*EVPN multihoming* (EVPN-MH) provides support for all-active server redundancy. It is a standards-based replacement for MLAG in data centers deploying Clos topologies. Replacing MLAG provides these benefits:

- Eliminates the need for peerlinks or inter-switch links between the top of rack switches
- Allows more than two TOR switches to participate in a redundancy group
- Provides a single BGP-EVPN control plane
- Allows multi-vendor interoperability

EVPN-MH uses {{<link url="#supported-evpn-route-types" text="BGP-EVPN type-1, type-2 and type-4 routes">}} for discovering Ethernet segments (ES) and for forwarding traffic to those Ethernet segments. The MAC and neighbor databases are synced between the Ethernet segment peers through these routes as well. An *{{<exlink url="https://tools.ietf.org/html/rfc7432#section-5" text="Ethernet segment">}}* is a group of switch links that are attached to the same server. Each Ethernet segment has an unique Ethernet segment ID (`es-id`) across the entire PoD.

{{%notice info%}}
EVPN-MH is only supported on Spectrum ASIC based switches.
{{% /notice %}}

Configuring EVPN-MH involves setting an Ethernet segment system MAC address (`es-sys-mac`) and a local Ethernet segment ID (`local-es-id`) on a static or LACP bond. These two parameters are used to automatically generate the unique MAC-based ESI value ({{<exlink url="https://tools.ietf.org/html/rfc7432#section-5" text="type-3">}}):

- The `es-sys-mac` is used for the LACP system identifier.
- The `es-id` configuration defines a local discriminator to uniquely enumerate each bond that shares the same es-sys-mac.
- The resulting 10-byte ESI value has the following format:

      03:MM:MM:MM:MM:MM:MM:XX:XX:XX

  where the MMs denote the 6-byte `es-sys-mac` and the XXs denote the 3-byte `es-id` value.

While you can specify a different `es-sys-mac` on different Ethernet segments attached to the same switch, the `es-sys-mac` must be the same on the downlinks attached to the same server.

{{%notice info%}}
When using Spectrum 2 or Spectrum 3 switches, an Ethernet segment can span more than two switches. Each Ethernet segment is a distinct redundancy group. However, when using Spectrum A1 switches, a maximum of two switches can participate in a redundancy group or Ethernet segment.
{{%/notice%}}

## Required and Supported Features

This section describes features that must be enabled in order to use EVPN multihoming, other supported features and features that are not supported.

### Required Features

To use EVPN-MH, you must enable the following features:

- {{<link url="VLAN-aware-Bridge-Mode" text="VLAN-aware bridge mode">}}
- {{<link url="Basic-Configuration/#arp-and-nd-suppression" text="ARP suppression">}}
- EVPN BUM traffic handling with {{<link title="EVPN BUM Traffic with PIM-SM" text="EVPN-PIM">}} on multihomed sites via Type-4/ESR routes, which includes split-horizon-filtering and designated forwarder election

{{%notice warning%}}
To use EVPN-MH, you must remove any MLAG configuration on the switch:
- Remove the `clag-id` from all interfaces in the `/etc/network/interfaces` file.
- Remove the peerlink interfaces in the `/etc/network/interfaces` file.
- Remove any existing `hwaddress` (from a Cumulus Linux 3.x MLAG configuration) or `address-virtual` (from a Cumulus Linux 4.x MLAG configuration) entries from all SVIs corresponding to a layer 3 VNI in the `/etc/network/interfaces` file.
- Remove any `clagd-vxlan-anycast-ip` configuration in the `/etc/network/interfaces` file.
- Reload the configuration with the `sudo ifreload` command.
{{%/notice%}}

### Other Supported Features

- Known unicast traffic multihoming via type-1/EAD (Ethernet auto discovery) routes and type-2 (non-zero ESI) routes. Includes all-active redundancy via aliasing and support for fast failover.
- {{<link url="LACP-Bypass">}} is supported.
  - When an EVPN-MH bond enters LACP bypass state, BGP stops advertising EVPN type-1 and type-4 routes for that bond. Split-horizon and designated forwarder filters are disabled.
  - When an EVPN-MH bond exits the LACP bypass state, BGP starts advertising EVPN type-1 and type-4 routes for that bond. Split-horizon and designated forwarder filters are enabled.
- EVI (*EVPN virtual instance*). Cumulus Linux supports VLAN-based service only, so the EVI is just a layer 2 VNI.
- Supported {{<exlink url="https://www.nvidia.com/en-us/networking/ethernet-switching/hardware-compatibility-list/" text="ASICs">}} include Mellanox Spectrum A1, Spectrum 2 and Spectrum 3.

### Supported EVPN Route Types

EVPN multihoming supports the following route types.

| Route Type | Description | RFC or Draft |
| ---------- | ----------- | ------------ |
| 1 | Ethernet auto-discovery (A-D) route | {{<exlink url="https://tools.ietf.org/html/rfc7432" text="RFC 7432">}} |
| 2 | MAC/IP advertisement route | {{<exlink url="https://tools.ietf.org/html/rfc7432" text="RFC 7432">}} |
| 3 | Inclusive multicast route | {{<exlink url="https://tools.ietf.org/html/rfc7432" text="RFC 7432">}} |
| 4 | Ethernet segment route | {{<exlink url="https://tools.ietf.org/html/rfc7432" text="RFC 7432">}} |
| 5 | IP prefix route | {{<exlink url="https://tools.ietf.org/html/draft-ietf-bess-evpn-prefix-advertisement-04" text="draft-ietf-bess-evpn-prefix-advertisement-04">}} |

### Unsupported Features

{{% notice note %}}
- EVPN MH cannot coexist in an EVPN network with Broadcom based switches in an MLAG configuration.
- EVPN-MH is not supported in mixed Broadcom-Spectrum networks.
- EVPN and MLAG can coexist in networks with all Spectrum based switches.
{{% /notice %}}

The following features are not supported with EVPN-MH:

- {{<link url="Traditional-Bridge-Mode" text="Traditional bridge mode">}}
- {{<link url="Inter-subnet-Routing/#asymmetric-routing" text="Distributed asymmetric routing">}}
- Head-end replication; use {{<link title="EVPN BUM Traffic with PIM-SM" text="EVPN-PIM">}} for BUM traffic handling.

## Basic Configuration

To configure EVPN-MH, enable the `evpn.multihoming.enable` variable in `switchd.conf`. Then, specify the following required settings:

- The Ethernet segment ID (`es-id`)
- The Ethernet segment system MAC address (`es-sys-mac`)

These settings are applied to interfaces, typically bonds.

An Ethernet segment configuration has these characteristics:

- The `es-id` is a 24-bit integer (1-16777215).
- Each interface (bond) needs its own `es-id`.
- Static and LACP bonds can be associated with an `es-id`.

A *designated forwarder* (DF) is elected for each Ethernet segment. The DF is responsible for forwarding flooded traffic received through the VXLAN overlay to the locally attached Ethernet segment. NVIDIA recommends you specify a preference (using the `es-df-pref` option) on an Ethernet segment for the DF election, as this leads to predictable failure scenarios. The EVPN VTEP with the highest `es-df-pref` setting becomes the DF. The `es-df-pref` setting defaults to _32767_.

NCLU generates the EVPN-MH configuration and reloads FRR and `ifupdown2`. The configuration appears in both the `/etc/network/interfaces` file and in `/etc/frr/frr.conf` file.

{{%notice note%}}
When EVPN-MH is enabled, all SVI MAC addresses are advertised as type 2 routes. You no longer need to configure a unique SVI IP address or configure the BGP EVPN address family with `advertise-svi-ip`.
{{%/notice%}}

### Enable EVPN-MH in switchd

To enable EVPN-MH in `switchd`, set the `evpn.multihoming.enable` variable in `switchd.conf` to _TRUE_, then restart the `switchd` service. The variable is disabled by default.

```
cumulus@switch:~$ sudo nano /etc/cumulus/switchd.conf
...

evpn.multihoming.enable = TRUE

...

cumulus@switch:~$ sudo systemctl restart switchd.service
```

### Configure the EVPN-MH Bonds

To configure bond interfaces for EVPN multihoming, run commands similar to the following:

{{<tabs "bond config">}}
{{<tab "NCLU Commands">}}

```
cumulus@switch:~$ net add bond hostbond1 bond slaves swp1
cumulus@switch:~$ net add bond hostbond2 bond slaves swp2
cumulus@switch:~$ net add bond hostbond3 bond slaves swp3
cumulus@switch:~$ net add bond hostbond1 evpn mh es-id 1
cumulus@switch:~$ net add bond hostbond2 evpn mh es-id 2
cumulus@switch:~$ net add bond hostbond3 evpn mh es-id 3
cumulus@switch:~$ net add bond hostbond1-3 evpn mh es-sys-mac 44:38:39:ff:ff:01
cumulus@switch:~$ net add bond hostbond1-3 evpn mh es-df-pref 50000
cumulus@switch:~$ net commit
```

{{</tab>}}
{{<tab "vtysh Commands">}}

```
cumulus@leaf01:~$ sudo vtysh
...
leaf01# configure terminal
leaf01(config)# interface bond1
leaf01(config-if)# evpn mh es-df-pref 50000
leaf01(config-if)# evpn mh es-id 1
leaf01(config-if)# evpn mh es-sys-mac 44:38:39:BE:EF:AA
leaf01(config-if)# exit
leaf01(config)# interface bond2
leaf01(config-if)# evpn mh es-df-pref 50000
leaf01(config-if)# evpn mh es-id 2
leaf01(config-if)# evpn mh es-sys-mac 44:38:39:BE:EF:AA
leaf01(config-if)# exit
leaf01(config)# interface bond3
leaf01(config-if)# evpn mh es-df-pref 50000
leaf01(config-if)# evpn mh es-id 3
leaf01(config-if)# evpn mh es-sys-mac 44:38:39:BE:EF:AA
leaf01(config-if)# exit
leaf01(config)# write memory
leaf01(config)# exit
leaf01# exit
cumulus@leaf01:~$
```

{{</tab>}}
{{</tabs>}}

The NCLU commands create the following configuration in the `/etc/network/interfaces` file. If you are editing the `/etc/network/interfaces` file directly, apply a configuration like the following:

```
interface bond1
  bond-slaves swp1
  es-sys-mac 44:38:39:BE:EF:AA

interface bond2
  bond-slaves swp2
  es-sys-mac 44:38:39:BE:EF:AA

interface bond3
  bond-slaves swp3
  es-sys-mac 44:38:39:BE:EF:AA
```

These commands also create the following configuration in the `/etc/frr/frr.conf` file.

```
!
interface bond1
 evpn mh es-df-pref 50000
 evpn mh es-id 1
 evpn mh es-sys-mac 44:38:39:BE:EF:AA
!
interface bond2
 evpn mh es-df-pref 50000
 evpn mh es-id 2
 evpn mh es-sys-mac 44:38:39:BE:EF:AA
!
interface bond3
 evpn mh es-df-pref 50000
 evpn mh es-id 3
 evpn mh es-sys-mac 44:38:39:BE:EF:AA
!
```

## Optional Configuration

### Global Settings

You can set these global settings for EVPN-MH:
- `mac-holdtime` specifies the duration for which a switch maintains SYNC MAC entries after the EVPN type-2 route of the Ethernet segment peer is deleted. During this time, the switch attempts to independently establish reachability of the MAC address on the local Ethernet segment. The hold time can be between 0 and 86400 seconds. The default is 1080 seconds.
- `neigh-holdtime` specifies the duration for which a switch maintains SYNC neighbor entries after the EVPN type-2 route of the Ethernet segment peer is deleted. During this time, the switch attempts to independently establish reachability of the host on the local Ethernet segment. The hold time can be between 0 and 86400 seconds. The default is 1080 seconds.
- `redirect-off` disables fast failover of traffic destined to the access port via the VXLAN overlay. This only applies to Cumulus VX (fast failover is only supported on the ASIC).
- `startup-delay` specifies the duration for which a switch holds the Ethernet segment-bond in a protodown state after a reboot or process restart. This allows the initialization of the VXLAN overlay to complete. The delay can be between 0 and 216000 seconds. The default is 180 seconds.

To configure a MAC hold time for 1000 seconds, run the following commands:

{{<tabs "MAC hold time">}}
{{<tab "NCLU Commands">}}

```
cumulus@switch:~$ net add evpn mh mac-holdtime 1000
cumulus@switch:~$ net commit
```

{{</tab>}}
{{<tab "vtysh Commands">}}

```
cumulus@switch:~$ sudo vtysh
switch# configure terminal
switch(config)# evpn mh mac-holdtime 1000
switch(config)# exit
switch# write memory
```

{{</tab>}}
{{</tabs>}}

This creates the following configuration in the `/etc/frr/frr.conf` file:

```
evpn mh mac-holdtime 1200
```

To configure a neighbor hold time for 600 seconds, run the following commands:

{{<tabs "Neighbor hold time">}}
{{<tab "NCLU Commands">}}

```
cumulus@switch:~$ net add evpn mh neigh-holdtime 600
cumulus@switch:~$ net commit
```

{{</tab>}}
{{<tab "vtysh Commands">}}

```
cumulus@switch:~$ sudo vtysh
switch# configure terminal
switch(config)# evpn mh neigh-holdtime 600
switch(config)# exit
switch# write memory
```

{{</tab>}}
{{</tabs>}}

This creates the following configuration in the `/etc/frr/frr.conf` file:

```
evpn mh neigh-holdtime 600
```

To configure a startup delay for 1800 seconds, run the following commands:

{{<tabs "startup delay">}}
{{<tab "NCLU Commands">}}

```
cumulus@switch:~$ net add evpn mh startup-delay 1800
cumulus@switch:~$ net commit
```

{{</tab>}}
{{<tab "vtysh Commands">}}

```
cumulus@switch:~$ sudo vtysh
switch# configure terminal
switch(config)# evpn mh startup-delay 1800
switch(config)# exit
switch# write memory
```

{{</tab>}}
{{</tabs>}}

This creates the following configuration in the `/etc/frr/frr.conf` file:

```
evpn mh startup-delay 1800
```

### Enable Uplink Tracking

When all the uplinks go down, the VTEP loses connectivity to the VXLAN overlay. To prevent traffic loss in this state, the operational state of the uplinks is tracked. When all the uplinks are down, the Ethernet segment bonds on the switch are put into a protodown or error-disabled state. You can configure a link as an MH uplink to enable this tracking.

{{<tabs "upink tracking">}}
{{<tab "NCLU Commands">}}

```
cumulus@switch:~$ net add interface swp51-54 evpn mh uplink
cumulus@switch:~$ net add interface swp51-54 pim
cumulus@switch:~$ net commit
```

{{</tab>}}
{{<tab "vtysh Commands">}}

```
cumulus@leaf01:~$ sudo vtysh
...
leaf01# configure terminal
leaf01(config)# interface swp51
leaf01(config-if)# evpn mh uplink
leaf01(config-if)# ip pim
leaf01(config-if)# exit
leaf01(config)# interface swp52
leaf01(config-if)# evpn mh uplink
leaf01(config-if)# ip pim
leaf01(config-if)# exit
leaf01(config)# interface swp53
leaf01(config-if)# evpn mh uplink
leaf01(config-if)# ip pim
leaf01(config-if)# exit
leaf01(config)# interface swp54
leaf01(config-if)# evpn mh uplink
leaf01(config-if)# ip pim
leaf01(config-if)# exit
leaf01(config)# write memory
leaf01(config)# exit
leaf01# exit
cumulus@leaf01:~$
```

{{</tab>}}
{{</tabs>}}

These commands create the following configuration in the `/etc/frr/frr.conf` file:

```
...
!
interface swp51
 evpn mh uplink
 ip pim
!
interface swp52
 evpn mh uplink
 ip pim
!
interface swp53
 evpn mh uplink
 ip pim
!
interface swp54
 evpn mh uplink
 ip pim
!
...
```

### Enable FRR Debugging

You can add debug statements to the `/etc/frr/frr.conf` file to debug the Ethernet segments, routes and routing protocols (via Zebra).

{{<tabs "debug">}}
{{<tab "NCLU Commands">}}

To debug Ethernet segments and routes, use the `net add bgp debug evpn mh (es|route)` command. To debug the routing protocols, use `net add evpn mh debug zebra (es|mac|neigh|nh)`.

```
cumulus@switch:~$ net add bgp debug evpn mh es
cumulus@switch:~$ net add bgp debug evpn mh route
cumulus@switch:~$ net add evpn mh debug zebra
cumulus@switch:~$ net add evpn mh debug zebra es
cumulus@switch:~$ net add evpn mh debug zebra mac
cumulus@switch:~$ net add evpn mh debug zebra neigh
cumulus@switch:~$ net add evpn mh debug zebra nh
cumulus@switch:~$ net commit
```

{{</tab>}}
{{<tab "vtysh Commands">}}

```
cumulus@leaf01:~$ sudo vtysh
...
leaf01# configure terminal
leaf01(config)# debug bgp evpn mh es
leaf01(config)# debug bgp evpn mh route
leaf01(config)# debug bgp zebra
leaf01(config)# debug zebra evpn mh es
leaf01(config)# debug zebra evpn mh mac
leaf01(config)# debug zebra evpn mh neigh
leaf01(config)# debug zebra evpn mh nh
leaf01(config)# debug zebra vxlan
leaf01(config)# write memory
leaf01(config)# exit
leaf01# exit
cumulus@leaf01:~$
```

{{</tab>}}
{{</tabs>}}

These commands create the following configuration in the `/etc/frr/frr.conf` file:

```
cumulus@switch:~$ cat /etc/frr/frr.conf
frr version 7.4+cl4u1
frr defaults datacenter

...

!
debug bgp evpn mh es
debug bgp evpn mh route
debug bgp zebra
debug zebra evpn mh es
debug zebra evpn mh mac
debug zebra evpn mh neigh
debug zebra evpn mh nh
debug zebra vxlan
!
...
```

### Fast Failover

When an Ethernet segment link goes down, the attached VTEP notifies all other VTEPs via a single EAD-ES withdraw. This is done by way of an Ethernet segment bond redirect.

Fast failover is also triggered by:

- Rebooting a leaf switch or VTEP.
- Uplink failure. When all uplinks are down, the Ethernet segment bonds on the switch are protodowned or error disabled.

### Disable Next Hop Group Sharing in the ASIC

Container sharing for both layer 2 and layer 3 next hop groups is enabled by default when EVPN-MH is configured. These settings are stored in the `evpn.multihoming.shared_l2_groups` and `evpn.multihoming.shared_l3_groups` variables.

Disabling container sharing allows for faster failover when an Ethernet segment link flaps.

To disable either setting, edit `switchd.conf`, set the variable to _FALSE_, then restart the `switchd` service. For example, to disable container sharing for layer 3 next hop groups, do the following:

```
cumulus@switch:~$ sudo nano /etc/cumulus/switchd.conf
...

evpn.multihoming.shared_l3_groups = FALSE

...

cumulus@switch:~$ sudo systemctl restart switchd.service
```

### Disable EAD-per-EVI Route Advertisements

{{<exlink url="https://tools.ietf.org/html/rfc7432" text="RFC 7432">}} requires type-1/EAD (Ethernet Auto-discovery) routes to be advertised two ways:

- As EAD-per-ES (Ethernet Auto-discovery per Ethernet segment) routes
- As EAD-per-EVI (Ethernet Auto-discovery per EVPN instance) routes

Some third party switch vendors don't advertise EAD-per-EVI routes; they only advertise EAD-per-ES routes. To interoperate with these vendors, you need to disable EAD-per-EVI route advertisements.

To remove the dependency on EAD-per-EVI routes and activate the VTEP upon receiving the EAD-per-ES route, run:

```
cumulus@switch:~$ net add bgp l2vpn evpn disable-ead-evi-rx
cumulus@switch:~$ net commit
```

To suppress the advertisement of EAD-per-EVI routes, run:

```
cumulus@switch:~$ net add bgp l2vpn evpn disable-ead-evi-tx
cumulus@switch:~$ net commit
```

## Troubleshooting

You can use the following `net show` commands to troubleshoot your EVPN multihoming configuration.

### Show Ethernet Segment Information

The `net show evpn es` command displays the Ethernet segments across all VNIs.

```
cumulus@switch:~$ net show evpn es
Type: L local, R remote, N non-DF
ESI                            Type ES-IF                 VTEPs
03:44:38:39:ff:ff:01:00:00:01  R    -                     172.0.0.22,172.0.0.23
03:44:38:39:ff:ff:01:00:00:02  LR   hostbond2             172.0.0.22,172.0.0.23
03:44:38:39:ff:ff:01:00:00:03  LR   hostbond3             172.0.0.22,172.0.0.23
03:44:38:39:ff:ff:01:00:00:05  L    hostbond1
03:44:38:39:ff:ff:02:00:00:01  R    -                     172.0.0.24,172.0.0.25,172.0.0.26
03:44:38:39:ff:ff:02:00:00:02  R    -                     172.0.0.24,172.0.0.25,172.0.0.26
03:44:38:39:ff:ff:02:00:00:03  R    -                     172.0.0.24,172.0.0.25,172.0.0.26
```

### Show Ethernet Segment per VNI Information

The `net show evpn es-evi` command displays the Ethernet segments learned for each VNI.

```
cumulus@switch:~$ net show evpn es-evi
Type: L local, R remote
VNI      ESI                            Type
...
1002     03:44:38:39:ff:ff:01:00:00:02  L
1002     03:44:38:39:ff:ff:01:00:00:03  L
1002     03:44:38:39:ff:ff:01:00:00:05  L
1001     03:44:38:39:ff:ff:01:00:00:02  L
1001     03:44:38:39:ff:ff:01:00:00:03  L
1001     03:44:38:39:ff:ff:01:00:00:05  L
...
```

### Show BGP Ethernet Segment Information

The `net show bgp l2vpn evpn es` command displays the Ethernet segments across all VNIs learned via type-1 and type-4 routes.

```
cumulus@switch:~$ net show bgp l2vpn evpn es
ES Flags: L local, R remote, I inconsistent
VTEP Flags: E ESR/Type-4, A active nexthop
ESI                            Flags RD                    #VNIs    VTEPs
03:44:38:39:ff:ff:01:00:00:01  LR    172.0.0.9:3            10       172.0.0.10(EA),172.0.0.11(EA)
03:44:38:39:ff:ff:01:00:00:02  LR    172.0.0.9:4            10       172.0.0.10(EA),172.0.0.11(EA)
03:44:38:39:ff:ff:01:00:00:03  LR    172.0.0.9:5            10       172.0.0.10(EA),172.0.0.11(EA)
cumulus@switch:~$
```

### Show BGP Ethernet Segment per VNI Information

The `net show bgp l2vpn evpn es-evi` command displays the Ethernet segments per VNI learned via type-1 and type-4 routes.

```
cumulus@switch:~$ net show bgp l2vpn evpn es-evi
Flags: L local, R remote, I inconsistent
VTEP-Flags: E EAD-per-ES, V EAD-per-EVI
VNI      ESI                            Flags VTEPs
...
1002     03:44:38:39:ff:ff:01:00:00:01  R     172.0.0.22(EV),172.0.0.23(EV)
1002     03:44:38:39:ff:ff:01:00:00:02  LR    172.0.0.22(EV),172.0.0.23(EV)
1002     03:44:38:39:ff:ff:01:00:00:03  LR    172.0.0.22(EV),172.0.0.23(EV)
1002     03:44:38:39:ff:ff:01:00:00:05  L  
1002     03:44:38:39:ff:ff:02:00:00:01  R     172.0.0.24(EV),172.0.0.25(EV),172.0.0.26(EV)
1002     03:44:38:39:ff:ff:02:00:00:02  R     172.0.0.24(EV),172.0.0.25(EV),172.0.0.26(EV)
1002     03:44:38:39:ff:ff:02:00:00:03  R     172.0.0.24(EV),172.0.0.25(EV),172.0.0.26(EV)
1001     03:44:38:39:ff:ff:01:00:00:01  R     172.0.0.22(EV),172.0.0.23(EV)
1001     03:44:38:39:ff:ff:01:00:00:02  LR    172.0.0.22(EV),172.0.0.23(EV)
1001     03:44:38:39:ff:ff:01:00:00:03  LR    172.0.0.22(EV),172.0.0.23(EV)
1001     03:44:38:39:ff:ff:01:00:00:05  L  
1001     03:44:38:39:ff:ff:02:00:00:01  R     172.0.0.24(EV),172.0.0.25(EV),172.0.0.26(EV)
1001     03:44:38:39:ff:ff:02:00:00:02  R     172.0.0.24(EV),172.0.0.25(EV),172.0.0.26(EV)
1001     03:44:38:39:ff:ff:02:00:00:03  R     172.0.0.24(EV),172.0.0.25(EV),172.0.0.26(EV)
...
cumulus@switch:~$
```

### Show EAD Route Types

You can use the `net show bgp l2vpn evpn route` command to view type-1 EAD routes. Just include the `ead` route type option.

```
cumulus@switch:~$ net show bgp evpn l2vpn route type ead
BGP table version is 30, local router ID is 172.16.0.21
Status codes: s suppressed, d damped, h history, * valid, > best, i - internal
Origin codes: i - IGP, e - EGP, ? - incomplete
EVPN type-1 prefix: [1]:[ESI]:[EthTag]:[IPlen]:[VTEP-IP]
EVPN type-2 prefix: [2]:[EthTag]:[MAClen]:[MAC]:[IPlen]:[IP]
EVPN type-3 prefix: [3]:[EthTag]:[IPlen]:[OrigIP]
EVPN type-4 prefix: [4]:[ESI]:[IPlen]:[OrigIP]
EVPN type-5 prefix: [5]:[EthTag]:[IPlen]:[IP]

   Network          Next Hop            Metric LocPrf Weight Path
                    Extended Community
Route Distinguisher: 172.16.0.21:2
*> [1]:[0]:[03:44:38:39:ff:ff:01:00:00:01]:[128]:[0.0.0.0]
                    172.16.0.21                          32768 i
                    ET:8 RT:5556:1005
*> [1]:[0]:[03:44:38:39:ff:ff:01:00:00:02]:[128]:[0.0.0.0]
                    172.16.0.21                          32768 i
                    ET:8 RT:5556:1005
*> [1]:[0]:[03:44:38:39:ff:ff:01:00:00:03]:[128]:[0.0.0.0]
                    172.16.0.21                          32768 i
                    ET:8 RT:5556:1005

...

Displayed 198 prefixes (693 paths) (of requested type)
cumulus@switch:~$
```

## Example Configuration

The following example uses the topology illustrated here. It shows one rack for simplicity, but multiple racks can be added to this topology.

{{<img src="/images/cumulus-linux/EVPN-MH-example-config-citc.png">}}

### Configuration Commands

{{< tabs "TabID635 ">}}
{{<tab "NCLU Commands">}}

{{< tabs "TabID638 ">}}
{{< tab "leaf01 ">}}

```
cumulus@leaf01:~$ net add loopback lo ip address 10.10.10.1/32
cumulus@leaf01:~$ net add interface swp1-3,swp51-52
cumulus@leaf01:~$ net add bond bond1 bond slaves swp1
cumulus@leaf01:~$ net add bond bond2 bond slaves swp2
cumulus@leaf01:~$ net add bond bond3 bond slaves swp3
cumulus@leaf01:~$ net add interface swp1 alias bond member of bond1
cumulus@leaf01:~$ net add interface swp2 alias bond member of bond2
cumulus@leaf01:~$ net add interface swp3 alias bond member of bond3
cumulus@leaf01:~$ net add interface swp51-52 alias to spine
cumulus@leaf01:~$ net add bond bond1 bridge access 10
cumulus@leaf01:~$ net add bond bond2 bridge access 20
cumulus@leaf01:~$ net add bond bond3 bridge access 30
cumulus@leaf01:~$ net add bond bond1-3 bond lacp-bypass-allow
cumulus@leaf01:~$ net add bond bond1-3 mtu 9000
cumulus@leaf01:~$ net add bond bond1-3 stp bpduguard
cumulus@leaf01:~$ net add bond bond1-3 stp portadminedge
cumulus@leaf01:~$ net add bridge bridge ports bond1,bond2,bond3
cumulus@leaf01:~$ net add loopback lo pim
cumulus@leaf01:~$ net add loopback lo pim use-source 10.10.10.1
cumulus@leaf01:~$ net add pim rp 10.10.100.100 224.0.0.0/4
cumulus@leaf01:~$ net add pim ecmp
cumulus@leaf01:~$ net add pim keep-alive-timer 3600
cumulus@leaf01:~$ net add interface swp51-52 pim
cumulus@leaf01:~$ net add interface swp51-52 evpn mh uplink
cumulus@leaf01:~$ net add evpn mh startup-delay 10
cumulus@leaf01:~$ net add bond bond1 evpn mh es-df-pref 50000
cumulus@leaf01:~$ net add bond bond2 evpn mh es-df-pref 50000
cumulus@leaf01:~$ net add bond bond3 evpn mh es-df-pref 50000
cumulus@leaf01:~$ net add bond bond1 evpn mh es-sys-mac 44:38:39:BE:EF:AA
cumulus@leaf01:~$ net add bond bond2 evpn mh es-sys-mac 44:38:39:BE:EF:AA
cumulus@leaf01:~$ net add bond bond3 evpn mh es-sys-mac 44:38:39:BE:EF:AA
cumulus@leaf01:~$ net add bond bond1 evpn mh es-id 1
cumulus@leaf01:~$ net add bond bond2 evpn mh es-id 2
cumulus@leaf01:~$ net add bond bond3 evpn mh es-id 3
cumulus@leaf01:~$ net add vrf RED vni 4001
cumulus@leaf01:~$ net add vrf BLUE vni 4002
cumulus@leaf01:~$ net add vxlan vni10 vxlan id 10
cumulus@leaf01:~$ net add vxlan vni20 vxlan id 20
cumulus@leaf01:~$ net add vxlan vni30 vxlan id 30
cumulus@leaf01:~$ net add vxlan vniBLUE vxlan id 4002
cumulus@leaf01:~$ net add vxlan vniRED vxlan id 4001
cumulus@leaf01:~$ net add bridge bridge ports vni10,vni20,vni30,vniRED,vniBLUE
cumulus@leaf01:~$ net add bridge bridge vids 10,20,30,4001-4002
cumulus@leaf01:~$ net add bridge bridge vlan-aware
cumulus@leaf01:~$ net add vlan 10 ip address 10.1.10.2/24
cumulus@leaf01:~$ net add vlan 10 ip address-virtual 00:00:00:00:00:10 10.1.10.1/24
cumulus@leaf01:~$ net add vlan 10 vlan-id 10
cumulus@leaf01:~$ net add vlan 10 vlan-raw-device bridge
cumulus@leaf01:~$ net add vlan 10 vrf RED
cumulus@leaf01:~$ net add vlan 20 ip address 10.1.20.2/24
cumulus@leaf01:~$ net add vlan 20 ip address-virtual 00:00:00:00:00:20 10.1.20.1/24
cumulus@leaf01:~$ net add vlan 20 vlan-id 20
cumulus@leaf01:~$ net add vlan 20 vlan-raw-device bridge
cumulus@leaf01:~$ net add vlan 20 vrf RED
cumulus@leaf01:~$ net add vlan 30 ip address 10.1.30.2/24
cumulus@leaf01:~$ net add vlan 30 ip address-virtual 00:00:00:00:00:30 10.1.30.1/24
cumulus@leaf01:~$ net add vlan 30 vlan-id 30
cumulus@leaf01:~$ net add vlan 30 vlan-raw-device bridge
cumulus@leaf01:~$ net add vlan 30 vrf BLUE
cumulus@leaf01:~$ net add vlan 4001 hwaddress 44:38:39:BE:EF:AA
cumulus@leaf01:~$ net add vlan 4001 vlan-id 4001
cumulus@leaf01:~$ net add vlan 4001 vlan-raw-device bridge
cumulus@leaf01:~$ net add vlan 4001 vrf RED
cumulus@leaf01:~$ net add vlan 4002 hwaddress 44:38:39:BE:EF:AA
cumulus@leaf01:~$ net add vlan 4002 vlan-id 4002
cumulus@leaf01:~$ net add vlan 4002 vlan-raw-device bridge
cumulus@leaf01:~$ net add vlan 4002 vrf BLUE
cumulus@leaf01:~$ net add vrf BLUE,mgmt,RED vrf-table auto
cumulus@leaf01:~$ net add vxlan vni10 bridge access 10
cumulus@leaf01:~$ net add vxlan vni10 vxlan mcastgrp 224.0.0.10
cumulus@leaf01:~$ net add vxlan vni10,20,30,vniBLUE,vniRED bridge arp-nd-suppress on
cumulus@leaf01:~$ net add vxlan vni10,20,30,vniBLUE,vniRED bridge learning off
cumulus@leaf01:~$ net add vxlan vni10,20,30,vniBLUE,vniRED stp bpduguard
cumulus@leaf01:~$ net add vxlan vni10,20,30,vniBLUE,vniRED stp portbpdufilter
cumulus@leaf01:~$ net add vxlan vni20 bridge access 20
cumulus@leaf01:~$ net add vxlan vni20 vxlan mcastgrp 224.0.0.20
cumulus@leaf01:~$ net add vxlan vni30 bridge access 30
cumulus@leaf01:~$ net add vxlan vni30 vxlan mcastgrp 224.0.0.30
cumulus@leaf01:~$ net add vxlan vniBLUE bridge access 4002
cumulus@leaf01:~$ net add vxlan vniRED bridge access 4001
cumulus@leaf01:~$ net add loopback lo vxlan local-tunnelip 10.10.10.1
cumulus@leaf01:~$ net add routing route 0.0.0.0/0 192.168.200.1 vrf mgmt
cumulus@leaf01:~$ net add bgp autonomous-system 65101
cumulus@leaf01:~$ net add bgp router-id 10.10.10.1
cumulus@leaf01:~$ net add bgp neighbor underlay peer-group
cumulus@leaf01:~$ net add bgp neighbor underlay remote-as external
cumulus@leaf01:~$ net add bgp neighbor swp51 interface peer-group underlay
cumulus@leaf01:~$ net add bgp neighbor swp52 interface peer-group underlay
cumulus@leaf01:~$ net add bgp ipv4 unicast redistribute connected
cumulus@leaf01:~$ net add bgp l2vpn evpn neighbor underlay activate
cumulus@leaf01:~$ net add bgp l2vpn evpn advertise-all-vni
cumulus@leaf01:~$ net add bgp vrf RED autonomous-system 65101
cumulus@leaf01:~$ net add bgp vrf RED router-id 10.10.10.1
cumulus@leaf01:~$ net add bgp vrf RED ipv4 unicast redistribute connected
cumulus@leaf01:~$ net add bgp vrf RED l2vpn evpn  advertise ipv4 unicast
cumulus@leaf01:~$ net add bgp vrf BLUE autonomous-system 65101
cumulus@leaf01:~$ net add bgp vrf BLUE router-id 10.10.10.1
cumulus@leaf01:~$ net add bgp vrf BLUE ipv4 unicast redistribute connected
cumulus@leaf01:~$ net add bgp vrf BLUE l2vpn evpn advertise ipv4 unicast
cumulus@leaf01:~$ net commit
```

{{</tab>}}
{{<tab "leaf02">}}

```
cumulus@leaf02:~$ net add loopback lo ip address 10.10.10.2/32
cumulus@leaf02:~$ net add interface swp1-3,swp51-52
cumulus@leaf02:~$ net add bond bond1 bond slaves swp1
cumulus@leaf02:~$ net add bond bond2 bond slaves swp2
cumulus@leaf02:~$ net add bond bond3 bond slaves swp3
cumulus@leaf02:~$ net add interface swp1 alias bond member of bond1
cumulus@leaf02:~$ net add interface swp2 alias bond member of bond2
cumulus@leaf02:~$ net add interface swp3 alias bond member of bond3
cumulus@leaf02:~$ net add interface swp51-52 alias to spine
cumulus@leaf02:~$ net add bond bond1 bridge access 10
cumulus@leaf02:~$ net add bond bond2 bridge access 20
cumulus@leaf02:~$ net add bond bond3 bridge access 30
cumulus@leaf02:~$ net add bond bond1-3 bond lacp-bypass-allow
cumulus@leaf02:~$ net add bond bond1-3 mtu 9000
cumulus@leaf02:~$ net add bond bond1-3 stp bpduguard
cumulus@leaf02:~$ net add bond bond1-3 stp portadminedge
cumulus@leaf02:~$ net add bridge bridge ports bond1,bond2,bond3
cumulus@leaf02:~$ net add loopback lo pim
cumulus@leaf02:~$ net add loopback lo pim use-source 10.10.10.2
cumulus@leaf02:~$ net add pim rp 10.10.100.100 224.0.0.0/4
cumulus@leaf02:~$ net add pim ecmp
cumulus@leaf02:~$ net add pim keep-alive-timer 3600
cumulus@leaf02:~$ net add interface swp51-52 pim
cumulus@leaf02:~$ net add interface swp51-52 evpn mh uplink
cumulus@leaf02:~$ net add evpn mh startup-delay 10
cumulus@leaf02:~$ net add bond bond1 evpn mh es-df-pref 50000
cumulus@leaf02:~$ net add bond bond2 evpn mh es-df-pref 50000
cumulus@leaf02:~$ net add bond bond3 evpn mh es-df-pref 50000
cumulus@leaf02:~$ net add bond bond1 evpn mh es-sys-mac 44:38:39:BE:EF:AA
cumulus@leaf02:~$ net add bond bond2 evpn mh es-sys-mac 44:38:39:BE:EF:AA
cumulus@leaf02:~$ net add bond bond3 evpn mh es-sys-mac 44:38:39:BE:EF:AA
cumulus@leaf02:~$ net add bond bond1 evpn mh es-id 1
cumulus@leaf02:~$ net add bond bond2 evpn mh es-id 2
cumulus@leaf02:~$ net add bond bond3 evpn mh es-id 3
cumulus@leaf02:~$ net add vrf RED vni 4001
cumulus@leaf02:~$ net add vrf BLUE vni 4002
cumulus@leaf02:~$ net add vxlan vni10 vxlan id 10
cumulus@leaf02:~$ net add vxlan vni20 vxlan id 20
cumulus@leaf02:~$ net add vxlan vni30 vxlan id 30
cumulus@leaf02:~$ net add vxlan vniBLUE vxlan id 4002
cumulus@leaf02:~$ net add vxlan vniRED vxlan id 4001
cumulus@leaf02:~$ net add bridge bridge ports vni10,vni20,vni30,vniRED,vniBLUE
cumulus@leaf02:~$ net add bridge bridge vids 10,20,30,4001-4002
cumulus@leaf02:~$ net add bridge bridge vlan-aware
cumulus@leaf02:~$ net add vlan 10 ip address 10.1.10.2/24
cumulus@leaf02:~$ net add vlan 10 ip address-virtual 00:00:00:00:00:10 10.1.10.1/24
cumulus@leaf02:~$ net add vlan 10 vlan-id 10
cumulus@leaf02:~$ net add vlan 10 vlan-raw-device bridge
cumulus@leaf02:~$ net add vlan 10 vrf RED
cumulus@leaf02:~$ net add vlan 20 ip address 10.1.20.2/24
cumulus@leaf02:~$ net add vlan 20 ip address-virtual 00:00:00:00:00:20 10.1.20.1/24
cumulus@leaf02:~$ net add vlan 20 vlan-id 20
cumulus@leaf02:~$ net add vlan 20 vlan-raw-device bridge
cumulus@leaf02:~$ net add vlan 20 vrf RED
cumulus@leaf02:~$ net add vlan 30 ip address 10.1.30.2/24
cumulus@leaf02:~$ net add vlan 30 ip address-virtual 00:00:00:00:00:30 10.1.30.1/24
cumulus@leaf02:~$ net add vlan 30 vlan-id 30
cumulus@leaf02:~$ net add vlan 30 vlan-raw-device bridge
cumulus@leaf02:~$ net add vlan 30 vrf BLUE
cumulus@leaf02:~$ net add vlan 4001 hwaddress 44:38:39:BE:EF:AA
cumulus@leaf02:~$ net add vlan 4001 vlan-id 4001
cumulus@leaf02:~$ net add vlan 4001 vlan-raw-device bridge
cumulus@leaf02:~$ net add vlan 4001 vrf RED
cumulus@leaf02:~$ net add vlan 4002 hwaddress 44:38:39:BE:EF:AA
cumulus@leaf02:~$ net add vlan 4002 vlan-id 4002
cumulus@leaf02:~$ net add vlan 4002 vlan-raw-device bridge
cumulus@leaf02:~$ net add vlan 4002 vrf BLUE
cumulus@leaf02:~$ net add vrf BLUE,mgmt,RED vrf-table auto
cumulus@leaf02:~$ net add vxlan vni10 bridge access 10
cumulus@leaf02:~$ net add vxlan vni10 vxlan mcastgrp 224.0.0.10
cumulus@leaf02:~$ net add vxlan vni10,20,30,vniBLUE,vniRED bridge arp-nd-suppress on
cumulus@leaf02:~$ net add vxlan vni10,20,30,vniBLUE,vniRED bridge learning off
cumulus@leaf02:~$ net add vxlan vni10,20,30,vniBLUE,vniRED stp bpduguard
cumulus@leaf02:~$ net add vxlan vni10,20,30,vniBLUE,vniRED stp portbpdufilter
cumulus@leaf02:~$ net add vxlan vni20 bridge access 20
cumulus@leaf02:~$ net add vxlan vni20 vxlan mcastgrp 224.0.0.20
cumulus@leaf02:~$ net add vxlan vni30 bridge access 30
cumulus@leaf02:~$ net add vxlan vni30 vxlan mcastgrp 224.0.0.30
cumulus@leaf02:~$ net add vxlan vniBLUE bridge access 4002
cumulus@leaf02:~$ net add vxlan vniRED bridge access 4001
cumulus@leaf02:~$ net add loopback lo vxlan local-tunnelip 10.10.10.2
cumulus@leaf02:~$ net add routing route 0.0.0.0/0 192.168.200.1 vrf mgmt
cumulus@leaf02:~$ net add bgp autonomous-system 65102
cumulus@leaf02:~$ net add bgp router-id 10.10.10.2
cumulus@leaf02:~$ net add bgp neighbor underlay peer-group
cumulus@leaf02:~$ net add bgp neighbor underlay remote-as external
cumulus@leaf02:~$ net add bgp neighbor swp51 interface peer-group underlay
cumulus@leaf02:~$ net add bgp neighbor swp52 interface peer-group underlay
cumulus@leaf02:~$ net add bgp ipv4 unicast redistribute connected
cumulus@leaf02:~$ net add bgp l2vpn evpn neighbor underlay activate
cumulus@leaf02:~$ net add bgp l2vpn evpn advertise-all-vni
cumulus@leaf02:~$ net add bgp vrf RED autonomous-system 65102
cumulus@leaf02:~$ net add bgp vrf RED router-id 10.10.10.2
cumulus@leaf02:~$ net add bgp vrf RED ipv4 unicast redistribute connected
cumulus@leaf02:~$ net add bgp vrf RED l2vpn evpn  advertise ipv4 unicast
cumulus@leaf02:~$ net add bgp vrf BLUE autonomous-system 65102
cumulus@leaf02:~$ net add bgp vrf BLUE router-id 10.10.10.2
cumulus@leaf02:~$ net add bgp vrf BLUE ipv4 unicast redistribute connected
cumulus@leaf02:~$ net add bgp vrf BLUE l2vpn evpn advertise ipv4 unicast
cumulus@leaf02:~$ net commit
```

{{</tab>}}
{{<tab "leaf03">}}

```
cumulus@leaf03:~$ net add loopback lo ip address 10.10.10.3/32
cumulus@leaf03:~$ net add interface swp1-3,swp51-52
cumulus@leaf03:~$ net add bond bond1 bond slaves swp1
cumulus@leaf03:~$ net add bond bond2 bond slaves swp2
cumulus@leaf03:~$ net add bond bond3 bond slaves swp3
cumulus@leaf03:~$ net add interface swp1 alias bond member of bond1
cumulus@leaf03:~$ net add interface swp2 alias bond member of bond2
cumulus@leaf03:~$ net add interface swp3 alias bond member of bond3
cumulus@leaf03:~$ net add interface swp51-52 alias to spine
cumulus@leaf03:~$ net add bond bond1 bridge access 10
cumulus@leaf03:~$ net add bond bond2 bridge access 20
cumulus@leaf03:~$ net add bond bond3 bridge access 30
cumulus@leaf03:~$ net add bond bond1-3 bond lacp-bypass-allow
cumulus@leaf03:~$ net add bond bond1-3 mtu 9000
cumulus@leaf03:~$ net add bond bond1-3 stp bpduguard
cumulus@leaf03:~$ net add bond bond1-3 stp portadminedge
cumulus@leaf03:~$ net add bridge bridge ports bond1,bond2,bond3
cumulus@leaf03:~$ net add loopback lo pim
cumulus@leaf03:~$ net add loopback lo pim use-source 10.10.10.3
cumulus@leaf03:~$ net add pim rp 10.10.100.100 224.0.0.0/4
cumulus@leaf03:~$ net add pim ecmp
cumulus@leaf03:~$ net add pim keep-alive-timer 3600
cumulus@leaf03:~$ net add interface swp51-52 pim
cumulus@leaf03:~$ net add interface swp51-52 evpn mh uplink
cumulus@leaf03:~$ net add evpn mh startup-delay 10
cumulus@leaf03:~$ net add bond bond1 evpn mh es-df-pref 50000
cumulus@leaf03:~$ net add bond bond2 evpn mh es-df-pref 50000
cumulus@leaf03:~$ net add bond bond3 evpn mh es-df-pref 50000
cumulus@leaf03:~$ net add bond bond1 evpn mh es-sys-mac 44:38:39:BE:EF:AA
cumulus@leaf03:~$ net add bond bond2 evpn mh es-sys-mac 44:38:39:BE:EF:AA
cumulus@leaf03:~$ net add bond bond3 evpn mh es-sys-mac 44:38:39:BE:EF:AA
cumulus@leaf03:~$ net add bond bond1 evpn mh es-id 1
cumulus@leaf03:~$ net add bond bond2 evpn mh es-id 2
cumulus@leaf03:~$ net add bond bond3 evpn mh es-id 3
cumulus@leaf03:~$ net add vrf RED vni 4001
cumulus@leaf03:~$ net add vrf BLUE vni 4002
cumulus@leaf03:~$ net add vxlan vni10 vxlan id 10
cumulus@leaf03:~$ net add vxlan vni20 vxlan id 20
cumulus@leaf03:~$ net add vxlan vni30 vxlan id 30
cumulus@leaf03:~$ net add vxlan vniBLUE vxlan id 4002
cumulus@leaf03:~$ net add vxlan vniRED vxlan id 4001
cumulus@leaf03:~$ net add bridge bridge ports vni10,vni20,vni30,vniRED,vniBLUE
cumulus@leaf03:~$ net add bridge bridge vids 10,20,30,4001-4002
cumulus@leaf03:~$ net add bridge bridge vlan-aware
cumulus@leaf03:~$ net add vlan 10 ip address 10.1.10.1/24
cumulus@leaf03:~$ net add vlan 10 ip address-virtual 00:00:00:00:00:10 10.1.10.1/24
cumulus@leaf03:~$ net add vlan 10 vlan-id 10
cumulus@leaf03:~$ net add vlan 10 vlan-raw-device bridge
cumulus@leaf03:~$ net add vlan 10 vrf RED
cumulus@leaf03:~$ net add vlan 20 ip address 10.1.20.1/24
cumulus@leaf03:~$ net add vlan 20 ip address-virtual 00:00:00:00:00:20 10.1.20.1/24
cumulus@leaf03:~$ net add vlan 20 vlan-id 20
cumulus@leaf03:~$ net add vlan 20 vlan-raw-device bridge
cumulus@leaf03:~$ net add vlan 20 vrf RED
cumulus@leaf03:~$ net add vlan 30 ip address 10.1.30.1/24
cumulus@leaf03:~$ net add vlan 30 ip address-virtual 00:00:00:00:00:30 10.1.30.1/24
cumulus@leaf03:~$ net add vlan 30 vlan-id 30
cumulus@leaf03:~$ net add vlan 30 vlan-raw-device bridge
cumulus@leaf03:~$ net add vlan 30 vrf BLUE
cumulus@leaf03:~$ net add vlan 4001 hwaddress 44:38:39:BE:EF:AA
cumulus@leaf03:~$ net add vlan 4001 vlan-id 4001
cumulus@leaf03:~$ net add vlan 4001 vlan-raw-device bridge
cumulus@leaf03:~$ net add vlan 4001 vrf RED
cumulus@leaf03:~$ net add vlan 4002 hwaddress 44:38:39:BE:EF:AA
cumulus@leaf03:~$ net add vlan 4002 vlan-id 4002
cumulus@leaf03:~$ net add vlan 4002 vlan-raw-device bridge
cumulus@leaf03:~$ net add vlan 4002 vrf BLUE
cumulus@leaf03:~$ net add vrf BLUE,mgmt,RED vrf-table auto
cumulus@leaf03:~$ net add vxlan vni10 bridge access 10
cumulus@leaf03:~$ net add vxlan vni10 vxlan mcastgrp 224.0.0.10
cumulus@leaf03:~$ net add vxlan vni10,20,30,vniBLUE,vniRED bridge arp-nd-suppress on
cumulus@leaf03:~$ net add vxlan vni10,20,30,vniBLUE,vniRED bridge learning off
cumulus@leaf03:~$ net add vxlan vni10,20,30,vniBLUE,vniRED stp bpduguard
cumulus@leaf03:~$ net add vxlan vni10,20,30,vniBLUE,vniRED stp portbpdufilter
cumulus@leaf03:~$ net add vxlan vni20 bridge access 20
cumulus@leaf03:~$ net add vxlan vni20 vxlan mcastgrp 224.0.0.20
cumulus@leaf03:~$ net add vxlan vni30 bridge access 30
cumulus@leaf03:~$ net add vxlan vni30 vxlan mcastgrp 224.0.0.30
cumulus@leaf03:~$ net add vxlan vniBLUE bridge access 4002
cumulus@leaf03:~$ net add vxlan vniRED bridge access 4001
cumulus@leaf03:~$ net add loopback lo vxlan local-tunnelip 10.10.10.3
cumulus@leaf03:~$ net add routing route 0.0.0.0/0 192.168.200.1 vrf mgmt
cumulus@leaf03:~$ net add bgp autonomous-system 65103
cumulus@leaf03:~$ net add bgp router-id 10.10.10.3
cumulus@leaf03:~$ net add bgp neighbor underlay peer-group
cumulus@leaf03:~$ net add bgp neighbor underlay remote-as external
cumulus@leaf03:~$ net add bgp neighbor swp51 interface peer-group underlay
cumulus@leaf03:~$ net add bgp neighbor swp52 interface peer-group underlay
cumulus@leaf03:~$ net add bgp ipv4 unicast redistribute connected
cumulus@leaf03:~$ net add bgp l2vpn evpn neighbor underlay activate
cumulus@leaf03:~$ net add bgp l2vpn evpn advertise-all-vni
cumulus@leaf03:~$ net add bgp vrf RED autonomous-system 65103
cumulus@leaf03:~$ net add bgp vrf RED router-id 10.10.10.3
cumulus@leaf03:~$ net add bgp vrf RED ipv4 unicast redistribute connected
cumulus@leaf03:~$ net add bgp vrf RED l2vpn evpn  advertise ipv4 unicast
cumulus@leaf03:~$ net add bgp vrf BLUE autonomous-system 65103
cumulus@leaf03:~$ net add bgp vrf BLUE router-id 10.10.10.3
cumulus@leaf03:~$ net add bgp vrf BLUE ipv4 unicast redistribute connected
cumulus@leaf03:~$ net add bgp vrf BLUE l2vpn evpn advertise ipv4 unicast
cumulus@leaf03:~$ net commit
```

{{</tab>}}
{{<tab "leaf04">}}

```
cumulus@leaf04:~$ net add loopback lo ip address 10.10.10.4/32
cumulus@leaf04:~$ net add interface swp1-3,swp51-52
cumulus@leaf04:~$ net add bond bond1 bond slaves swp1
cumulus@leaf04:~$ net add bond bond2 bond slaves swp2
cumulus@leaf04:~$ net add bond bond3 bond slaves swp3
cumulus@leaf04:~$ net add interface swp1 alias bond member of bond1
cumulus@leaf04:~$ net add interface swp2 alias bond member of bond2
cumulus@leaf04:~$ net add interface swp3 alias bond member of bond3
cumulus@leaf04:~$ net add interface swp51-52 alias to spine
cumulus@leaf04:~$ net add bond bond1 bridge access 10
cumulus@leaf04:~$ net add bond bond2 bridge access 20
cumulus@leaf04:~$ net add bond bond3 bridge access 30
cumulus@leaf04:~$ net add bond bond1-3 bond lacp-bypass-allow
cumulus@leaf04:~$ net add bond bond1-3 mtu 9000
cumulus@leaf04:~$ net add bond bond1-3 stp bpduguard
cumulus@leaf04:~$ net add bond bond1-3 stp portadminedge
cumulus@leaf04:~$ net add bridge bridge ports bond1,bond2,bond3
cumulus@leaf04:~$ net add loopback lo pim
cumulus@leaf04:~$ net add loopback lo pim use-source 10.10.10.4
cumulus@leaf04:~$ net add pim rp 10.10.100.100 224.0.0.0/4
cumulus@leaf04:~$ net add pim ecmp
cumulus@leaf04:~$ net add pim keep-alive-timer 3600
cumulus@leaf04:~$ net add interface swp51-52 pim
cumulus@leaf04:~$ net add interface swp51-52 evpn mh uplink
cumulus@leaf04:~$ net add evpn mh startup-delay 10
cumulus@leaf04:~$ net add bond bond1 evpn mh es-df-pref 50000
cumulus@leaf04:~$ net add bond bond2 evpn mh es-df-pref 50000
cumulus@leaf04:~$ net add bond bond3 evpn mh es-df-pref 50000
cumulus@leaf04:~$ net add bond bond1 evpn mh es-sys-mac 44:38:39:BE:EF:AA
cumulus@leaf04:~$ net add bond bond2 evpn mh es-sys-mac 44:38:39:BE:EF:AA
cumulus@leaf04:~$ net add bond bond3 evpn mh es-sys-mac 44:38:39:BE:EF:AA
cumulus@leaf04:~$ net add bond bond1 evpn mh es-id 1
cumulus@leaf04:~$ net add bond bond2 evpn mh es-id 2
cumulus@leaf04:~$ net add bond bond3 evpn mh es-id 3
cumulus@leaf04:~$ net add vrf RED vni 4001
cumulus@leaf04:~$ net add vrf BLUE vni 4002
cumulus@leaf04:~$ net add vxlan vni10 vxlan id 10
cumulus@leaf04:~$ net add vxlan vni20 vxlan id 20
cumulus@leaf04:~$ net add vxlan vni30 vxlan id 30
cumulus@leaf04:~$ net add vxlan vniBLUE vxlan id 4002
cumulus@leaf04:~$ net add vxlan vniRED vxlan id 4001
cumulus@leaf04:~$ net add bridge bridge ports vni10,vni20,vni30,vniRED,vniBLUE
cumulus@leaf04:~$ net add bridge bridge vids 10,20,30,4001-4002
cumulus@leaf04:~$ net add bridge bridge vlan-aware
cumulus@leaf04:~$ net add vlan 10 ip address 10.1.10.1/24
cumulus@leaf04:~$ net add vlan 10 ip address-virtual 00:00:00:00:00:10 10.1.10.1/24
cumulus@leaf04:~$ net add vlan 10 vlan-id 10
cumulus@leaf04:~$ net add vlan 10 vlan-raw-device bridge
cumulus@leaf04:~$ net add vlan 10 vrf RED
cumulus@leaf04:~$ net add vlan 20 ip address 10.1.20.1/24
cumulus@leaf04:~$ net add vlan 20 ip address-virtual 00:00:00:00:00:20 10.1.20.1/24
cumulus@leaf04:~$ net add vlan 20 vlan-id 20
cumulus@leaf04:~$ net add vlan 20 vlan-raw-device bridge
cumulus@leaf04:~$ net add vlan 20 vrf RED
cumulus@leaf04:~$ net add vlan 30 ip address 10.1.30.1/24
cumulus@leaf04:~$ net add vlan 30 ip address-virtual 00:00:00:00:00:30 10.1.30.1/24
cumulus@leaf04:~$ net add vlan 30 vlan-id 30
cumulus@leaf04:~$ net add vlan 30 vlan-raw-device bridge
cumulus@leaf04:~$ net add vlan 30 vrf BLUE
cumulus@leaf04:~$ net add vlan 4001 hwaddress 44:38:39:BE:EF:AA
cumulus@leaf04:~$ net add vlan 4001 vlan-id 4001
cumulus@leaf04:~$ net add vlan 4001 vlan-raw-device bridge
cumulus@leaf04:~$ net add vlan 4001 vrf RED
cumulus@leaf04:~$ net add vlan 4002 hwaddress 44:38:39:BE:EF:AA
cumulus@leaf04:~$ net add vlan 4002 vlan-id 4002
cumulus@leaf04:~$ net add vlan 4002 vlan-raw-device bridge
cumulus@leaf04:~$ net add vlan 4002 vrf BLUE
cumulus@leaf04:~$ net add vrf BLUE,mgmt,RED vrf-table auto
cumulus@leaf04:~$ net add vxlan vni10 bridge access 10
cumulus@leaf04:~$ net add vxlan vni10 vxlan mcastgrp 224.0.0.10
cumulus@leaf04:~$ net add vxlan vni10,20,30,vniBLUE,vniRED bridge arp-nd-suppress on
cumulus@leaf04:~$ net add vxlan vni10,20,30,vniBLUE,vniRED bridge learning off
cumulus@leaf04:~$ net add vxlan vni10,20,30,vniBLUE,vniRED stp bpduguard
cumulus@leaf04:~$ net add vxlan vni10,20,30,vniBLUE,vniRED stp portbpdufilter
cumulus@leaf04:~$ net add vxlan vni20 bridge access 20
cumulus@leaf04:~$ net add vxlan vni20 vxlan mcastgrp 224.0.0.20
cumulus@leaf04:~$ net add vxlan vni30 bridge access 30
cumulus@leaf04:~$ net add vxlan vni30 vxlan mcastgrp 224.0.0.30
cumulus@leaf04:~$ net add vxlan vniBLUE bridge access 4002
cumulus@leaf04:~$ net add vxlan vniRED bridge access 4001
cumulus@leaf04:~$ net add loopback lo vxlan local-tunnelip 10.10.10.4
cumulus@leaf04:~$ net add routing route 0.0.0.0/0 192.168.200.1 vrf mgmt
cumulus@leaf04:~$ net add bgp autonomous-system 65104
cumulus@leaf04:~$ net add bgp router-id 10.10.10.4
cumulus@leaf04:~$ net add bgp neighbor underlay peer-group
cumulus@leaf04:~$ net add bgp neighbor underlay remote-as external
cumulus@leaf04:~$ net add bgp neighbor swp51 interface peer-group underlay
cumulus@leaf04:~$ net add bgp neighbor swp52 interface peer-group underlay
cumulus@leaf04:~$ net add bgp ipv4 unicast redistribute connected
cumulus@leaf04:~$ net add bgp l2vpn evpn neighbor underlay activate
cumulus@leaf04:~$ net add bgp l2vpn evpn advertise-all-vni
cumulus@leaf04:~$ net add bgp vrf RED autonomous-system 65104
cumulus@leaf04:~$ net add bgp vrf RED router-id 10.10.10.4
cumulus@leaf04:~$ net add bgp vrf RED ipv4 unicast redistribute connected
cumulus@leaf04:~$ net add bgp vrf RED l2vpn evpn  advertise ipv4 unicast
cumulus@leaf04:~$ net add bgp vrf BLUE autonomous-system 65104
cumulus@leaf04:~$ net add bgp vrf BLUE router-id 10.10.10.4
cumulus@leaf04:~$ net add bgp vrf BLUE ipv4 unicast redistribute connected
cumulus@leaf04:~$ net add bgp vrf BLUE l2vpn evpn advertise ipv4 unicast
cumulus@leaf04:~$ net commit
```

{{</tab>}}
{{<tab "spine01">}}

```
cumulus@spine01:~$ net add loopback lo ip address 10.10.10.101/32
cumulus@spine01:~$ net add interface eth0 ip address 192.168.200.21/24
cumulus@spine01:~$ net add interface eth0 vrf mgmt
cumulus@spine01:~$ net add vrf mgmt ip address 127.0.0.1/8
cumulus@spine01:~$ net add vrf mgmt ipv6 address ::1/128
cumulus@spine01:~$ net add vrf mgmt vrf-table auto
cumulus@spine01:~$ net add loopback lo pim
cumulus@spine01:~$ net add loopback lo pim use-source 10.10.10.101
cumulus@spine01:~$ net add interface swp1-4 pim
cumulus@spine01:~$ net add interface swp1-4 alias to leaf
cumulus@spine01:~$ net add pim rp 10.10.100.100 224.0.0.0/4
cumulus@spine01:~$ net add pim ecmp
cumulus@spine01:~$ net add pim keep-alive-timer 3600
cumulus@spine01:~$ net add routing route 0.0.0.0/0 192.168.200.1 vrf mgmt
cumulus@spine01:~$ net add bgp autonomous-system 65100
cumulus@spine01:~$ net add bgp router-id 10.10.10.101
cumulus@spine01:~$ net add bgp neighbor underlay peer-group
cumulus@spine01:~$ net add bgp neighbor underlay remote-as external
cumulus@spine01:~$ net add bgp neighbor swp1 interface peer-group underlay
cumulus@spine01:~$ net add bgp neighbor swp2 interface peer-group underlay
cumulus@spine01:~$ net add bgp neighbor swp3 interface peer-group underlay
cumulus@spine01:~$ net add bgp neighbor swp4 interface peer-group underlay
cumulus@spine01:~$ net add bgp ipv4 unicast redistribute connected
cumulus@spine01:~$ net add bgp l2vpn evpn  neighbor underlay activate
cumulus@spine01:~$ net commit
```

{{</tab>}}
{{<tab "spine02">}}

**NCLU Commands**

```
cumulus@spine02:~$ net add loopback lo ip address 10.10.10.102/32
cumulus@spine02:~$ net add interface eth0 ip address 192.168.200.22/24
cumulus@spine02:~$ net add interface eth0 vrf mgmt
cumulus@spine02:~$ net add vrf mgmt ip address 127.0.0.1/8
cumulus@spine02:~$ net add vrf mgmt ipv6 address ::1/128
cumulus@spine02:~$ net add vrf mgmt vrf-table auto
cumulus@spine02:~$ net add loopback lo pim
cumulus@spine02:~$ net add loopback lo pim use-source 10.10.10.101
cumulus@spine02:~$ net add interface swp1-4 pim
cumulus@spine02:~$ net add interface swp1-4 alias to leaf
cumulus@spine02:~$ net add pim rp 10.10.100.100 224.0.0.0/4
cumulus@spine02:~$ net add pim ecmp
cumulus@spine02:~$ net add pim keep-alive-timer 3600
cumulus@spine02:~$ net add routing route 0.0.0.0/0 192.168.200.1 vrf mgmt
cumulus@spine02:~$ net add bgp autonomous-system 65100
cumulus@spine02:~$ net add bgp router-id 10.10.10.102
cumulus@spine02:~$ net add bgp neighbor underlay peer-group
cumulus@spine02:~$ net add bgp neighbor underlay remote-as external
cumulus@spine02:~$ net add bgp neighbor swp1 interface peer-group underlay
cumulus@spine02:~$ net add bgp neighbor swp2 interface peer-group underlay
cumulus@spine02:~$ net add bgp neighbor swp3 interface peer-group underlay
cumulus@spine02:~$ net add bgp neighbor swp4 interface peer-group underlay
cumulus@spine02:~$ net add bgp ipv4 unicast redistribute connected
cumulus@spine02:~$ net add bgp l2vpn evpn  neighbor underlay activate
cumulus@spine02:~$ net commit
```

{{</tab>}}
{{</tabs>}}

{{</tab>}}
{{<tab "Linux and vtysh Commands">}}

**Linux commands**:

1. On the leafs and spines, edit the `daemons` file to enable `bgp` and `pimd` (change *no* to *yes*).
2. On the leafs, spines, and servers, copy the configuration from the `/etc/network/interfaces` files show below.
3. On the leafs and spines, reload the new configuration with the `sudo ifreload -a` command.

**vtysh commands**:

{{< tabs "TabID1150 ">}}
{{<tab "leaf01">}}

```
cumulus@leaf01:~$ sudo vtysh
...
leaf01# configure terminal
leaf01(config)# vrf RED
leaf01(config-vrf)# vni 4001
leaf01(config-vrf)# exit-vrf
leaf01(config)# vrf BLUE
leaf01(config-vrf)# vni 4002
leaf01(config-vrf)# exit-vrf
leaf01(config)# interface bond1
leaf01(config-if)# evpn mh es-df-pref 50000
leaf01(config-if)# evpn mh es-id 1
leaf01(config-if)# evpn mh es-sys-mac 44:38:39:BE:EF:AA
leaf01(config-if)# exit
leaf01(config)# interface bond2
leaf01(config-if)# evpn mh es-df-pref 50000
leaf01(config-if)# evpn mh es-id 2
leaf01(config-if)# evpn mh es-sys-mac 44:38:39:BE:EF:AA
leaf01(config-if)# exit
leaf01(config)# interface bond3
leaf01(config-if)# evpn mh es-df-pref 50000
leaf01(config-if)# evpn mh es-id 3
leaf01(config-if)# evpn mh es-sys-mac 44:38:39:BE:EF:AA
leaf01(config-if)# exit
leaf01(config)# interface lo
leaf01(config-if)# ip pim
leaf01(config-if)# exit
leaf01(config)# interface swp51
leaf01(config-if)# evpn mh uplink
leaf01(config-if)# ip pim
leaf01(config-if)# exit
leaf01(config)# interface swp52
leaf01(config-if)# evpn mh uplink
leaf01(config-if)# ip pim
leaf01(config-if)# exit
leaf01(config)# ip route 0.0.0.0/0 192.168.200.1 vrf mgmt
leaf01(config)# ip pim rp 10.10.100.100 224.0.0.0/4
leaf01(config)# ip pim ecmp
leaf01(config)# ip pim keep-alive-timer 3600
leaf01(config)# router bgp 65101
leaf01(config-router)# bgp router-id 10.10.10.1
leaf01(config-router)# bgp bestpath as-path multipath-relax
leaf01(config-router)# neighbor underlay peer-group
leaf01(config-router)# neighbor underlay remote-as external
leaf01(config-router)# neighbor swp51 interface peer-group underlay
leaf01(config-router)# neighbor swp52 interface peer-group underlay
leaf01(config-router)# address-family ipv4 unicast
leaf01(config-router-af)# redistribute connected
leaf01(config-router-af)# neighbor underlay activate
leaf01(config-router-af)# exit-address-family
leaf01(config-router)# address-family l2vpn evpn
leaf01(config-router-af)# neighbor underlay activate
leaf01(config-router-af)# advertise-all-vni
leaf01(config-router-af)# exit-address-family
leaf01(config-router)# exit
leaf01(config)# router bgp 65101 vrf BLUE
leaf01(config-router)# bgp router-id 10.10.10.1
leaf01(config-router)# address-family ipv4 unicast
leaf01(config-router-af)# redistribute connected
leaf01(config-router-af)# exit-address-family
leaf01(config-router)# address-family l2vpn evpn
leaf01(config-router-af)# advertise ipv4 unicast
leaf01(config-router-af)# exit-address-family
leaf01(config-router)# exit
leaf01(config)# router bgp 65101 vrf RED
leaf01(config-router)# bgp router-id 10.10.10.1
leaf01(config-router)# address-family ipv4 unicast
leaf01(config-router-af)# redistribute connected
leaf01(config-router-af)# exit-address-family
leaf01(config-router)# address-family l2vpn evpn
leaf01(config-router-af)# advertise ipv4 unicast
leaf01(config-router-af)# exit-address-family
leaf01(config-router)# exit
leaf01# write memory
leaf01# exit
cumulus@leaf01:~$
```

{{</tab>}}
{{<tab "leaf02">}}

```
cumulus@leaf02:~$ sudo vtysh
leaf02# configure terminal
leaf02(config)# vrf RED
leaf02(config-vrf)# vni 4001
leaf02(config-vrf)# exit-vrf
leaf02(config)# vrf BLUE
leaf02(config-vrf)# vni 4002
leaf02(config-vrf)# exit-vrf
leaf02(config)# interface bond1
leaf02(config-if)# evpn mh es-df-pref 50000
leaf02(config-if)# evpn mh es-id 1
leaf02(config-if)# evpn mh es-sys-mac 44:38:39:BE:EF:AA
leaf02(config-if)# exit
leaf02(config)# interface bond2
leaf02(config-if)# evpn mh es-df-pref 50000
leaf02(config-if)# evpn mh es-id 2
leaf02(config-if)# evpn mh es-sys-mac 44:38:39:BE:EF:AA
leaf02(config-if)# exit
leaf02(config)# interface bond3
leaf02(config-if)# evpn mh es-df-pref 50000
leaf02(config-if)# evpn mh es-id 3
leaf02(config-if)# evpn mh es-sys-mac 44:38:39:BE:EF:AA
leaf02(config-if)# exit
leaf02(config)# interface lo
leaf02(config-if)# ip pim
leaf02(config-if)# exit
leaf02(config)# interface swp51
leaf02(config-if)# evpn mh uplink
leaf02(config-if)# ip pim
leaf02(config-if)# exit
leaf02(config)# interface swp52
leaf02(config-if)# evpn mh uplink
leaf02(config-if)# ip pim
leaf02(config-if)# exit
leaf02(config)# ip route 0.0.0.0/0 192.168.200.1 vrf mgmt
leaf02(config)# ip pim rp 10.10.100.100 224.0.0.0/4
leaf02(config)# ip pim ecmp
leaf02(config)# ip pim keep-alive-timer 3600
leaf02(config)# router bgp 65102
leaf02(config-router)# bgp router-id 10.10.10.2
leaf02(config-router)# bgp bestpath as-path multipath-relax
leaf02(config-router)# neighbor underlay peer-group
leaf02(config-router)# neighbor underlay remote-as external
leaf02(config-router)# neighbor swp51 interface peer-group underlay
leaf02(config-router)# neighbor swp52 interface peer-group underlay
leaf02(config-router)# address-family ipv4 unicast
leaf02(config-router-af)# redistribute connected
leaf02(config-router-af)# neighbor underlay activate
leaf02(config-router-af)# exit-address-family
leaf02(config-router)# address-family l2vpn evpn
leaf02(config-router-af)# neighbor underlay activate
leaf02(config-router-af)# advertise-all-vni
leaf02(config-router-af)# exit-address-family
leaf02(config-router)# exit
leaf02(config)# router bgp 65102 vrf BLUE
leaf02(config-router)# bgp router-id 10.10.10.2
leaf02(config-router)# address-family ipv4 unicast
leaf02(config-router-af)# redistribute connected
leaf02(config-router-af)# exit-address-family
leaf02(config-router)# address-family l2vpn evpn
leaf02(config-router-af)# advertise ipv4 unicast
leaf02(config-router-af)# exit-address-family
leaf02(config-router)# exit
leaf02(config)# router bgp 65102 vrf RED
leaf02(config-router)# bgp router-id 10.10.10.2
leaf02(config-router)# address-family ipv4 unicast
leaf02(config-router-af)# redistribute connected
leaf02(config-router-af)# exit-address-family
leaf02(config-router)# address-family l2vpn evpn
leaf02(config-router-af)# advertise ipv4 unicast
leaf02(config-router-af)# exit-address-family
leaf02(config-router)# exit
leaf02# write memory
leaf02# exit
cumulus@leaf02:~$
```

{{</tab>}}
{{<tab "leaf03">}}

```
cumulus@leaf03:~$ sudo vtysh
...
leaf03# configure terminal
leaf03(config)# vrf RED
leaf03(config-vrf)# vni 4001
leaf03(config-vrf)# exit-vrf
leaf03(config)# vrf BLUE
leaf03(config-vrf)# vni 4002
leaf03(config-vrf)# exit-vrf
leaf03(config)# interface bond1
leaf03(config-if)# evpn mh es-df-pref 50000
leaf03(config-if)# evpn mh es-id 1
leaf03(config-if)# evpn mh es-sys-mac 44:38:39:BE:EF:AA
leaf03(config-if)# exit
leaf03(config)# interface bond2
leaf03(config-if)# evpn mh es-df-pref 50000
leaf03(config-if)# evpn mh es-id 2
leaf03(config-if)# evpn mh es-sys-mac 44:38:39:BE:EF:AA
leaf03(config-if)# exit
leaf03(config)# interface bond3
leaf03(config-if)# evpn mh es-df-pref 50000
leaf03(config-if)# evpn mh es-id 3
leaf03(config-if)# evpn mh es-sys-mac 44:38:39:BE:EF:AA
leaf03(config-if)# exit
leaf03(config)# interface lo
leaf03(config-if)# ip pim
leaf03(config-if)# exit
leaf03(config)# interface swp51
leaf03(config-if)# evpn mh uplink
leaf03(config-if)# ip pim
leaf03(config-if)# exit
leaf03(config)# interface swp52
leaf03(config-if)# evpn mh uplink
leaf03(config-if)# ip pim
leaf03(config-if)# exit
leaf03(config)# ip route 0.0.0.0/0 192.168.200.1 vrf mgmt
leaf03(config)# ip pim rp 10.10.100.100 224.0.0.0/4
leaf03(config)# ip pim ecmp
leaf03(config)# ip pim keep-alive-timer 3600
leaf03(config)# router bgp 65103
leaf03(config-router)# bgp router-id 10.10.10.3
leaf03(config-router)# bgp bestpath as-path multipath-relax
leaf03(config-router)# neighbor underlay peer-group
leaf03(config-router)# neighbor underlay remote-as external
leaf03(config-router)# neighbor swp51 interface peer-group underlay
leaf03(config-router)# neighbor swp52 interface peer-group underlay
leaf03(config-router)# address-family ipv4 unicast
leaf03(config-router-af)# redistribute connected
leaf03(config-router-af)# neighbor underlay activate
leaf03(config-router-af)# exit-address-family
leaf03(config-router)# address-family l2vpn evpn
leaf03(config-router-af)# neighbor underlay activate
leaf03(config-router-af)# advertise-all-vni
leaf03(config-router-af)# exit-address-family
leaf03(config-router)# exit
leaf03(config)# router bgp 65103 vrf BLUE
leaf03(config-router)# bgp router-id 10.10.10.3
leaf03(config-router)# address-family ipv4 unicast
leaf03(config-router-af)# redistribute connected
leaf03(config-router-af)# exit-address-family
leaf03(config-router)# address-family l2vpn evpn
leaf03(config-router-af)# advertise ipv4 unicast
leaf03(config-router-af)# exit-address-family
leaf03(config-router)# exit
leaf03(config)# router bgp 65103 vrf RED
leaf03(config-router)# bgp router-id 10.10.10.3
leaf03(config-router)# address-family ipv4 unicast
leaf03(config-router-af)# redistribute connected
leaf03(config-router-af)# exit-address-family
leaf03(config-router)# address-family l2vpn evpn
leaf03(config-router-af)# advertise ipv4 unicast
leaf03(config-router-af)# exit-address-family
leaf03(config-router)# exit
leaf03# write memory
leaf03# exit
cumulus@leaf03:~$
```

{{</tab>}}
{{<tab "leaf04">}}

```
cumulus@leaf04:~$ sudo vtysh
...
leaf04# configure terminal
leaf04(config)# vrf RED
leaf04(config-vrf)# vni 4001
leaf04(config-vrf)# exit-vrf
leaf04(config)# vrf BLUE
leaf04(config-vrf)# vni 4002
leaf04(config-vrf)# exit-vrf
leaf04(config)# interface bond1
leaf04(config-if)# evpn mh es-df-pref 50000
leaf04(config-if)# evpn mh es-id 1
leaf04(config-if)# evpn mh es-sys-mac 44:38:39:BE:EF:AA
leaf04(config-if)# exit
leaf04(config)# interface bond2
leaf04(config-if)# evpn mh es-df-pref 50000
leaf04(config-if)# evpn mh es-id 2
leaf04(config-if)# evpn mh es-sys-mac 44:38:39:BE:EF:AA
leaf04(config-if)# exit
leaf04(config)# interface bond3
leaf04(config-if)# evpn mh es-df-pref 50000
leaf04(config-if)# evpn mh es-id 3
leaf04(config-if)# evpn mh es-sys-mac 44:38:39:BE:EF:AA
leaf04(config-if)# exit
leaf04(config)# interface lo
leaf04(config-if)# ip pim
leaf04(config-if)# exit
leaf04(config)# interface swp51
leaf04(config-if)# evpn mh uplink
leaf04(config-if)# ip pim
leaf04(config-if)# exit
leaf04(config)# interface swp52
leaf04(config-if)# evpn mh uplink
leaf04(config-if)# ip pim
leaf04(config-if)# exit
leaf04(config)# ip route 0.0.0.0/0 192.168.200.1 vrf mgmt
leaf04(config)# ip pim rp 10.10.100.100 224.0.0.0/4
leaf04(config)# ip pim ecmp
leaf04(config)# ip pim keep-alive-timer 3600
leaf04(config)# router bgp 65104
leaf04(config-router)# bgp router-id 10.10.10.4
leaf04(config-router)# bgp bestpath as-path multipath-relax
leaf04(config-router)# neighbor underlay peer-group
leaf04(config-router)# neighbor underlay remote-as external
leaf04(config-router)# neighbor swp51 interface peer-group underlay
leaf04(config-router)# neighbor swp52 interface peer-group underlay
leaf04(config-router)# address-family ipv4 unicast
leaf04(config-router-af)# redistribute connected
leaf04(config-router-af)# neighbor underlay activate
leaf04(config-router-af)# exit-address-family
leaf04(config-router)# address-family l2vpn evpn
leaf04(config-router-af)# neighbor underlay activate
leaf04(config-router-af)# advertise-all-vni
leaf04(config-router-af)# exit-address-family
leaf04(config-router)# exit
leaf04(config)# router bgp 65104 vrf BLUE
leaf04(config-router)# bgp router-id 10.10.10.4
leaf04(config-router)# address-family ipv4 unicast
leaf04(config-router-af)# redistribute connected
leaf04(config-router-af)# exit-address-family
leaf04(config-router)# address-family l2vpn evpn
leaf04(config-router-af)# advertise ipv4 unicast
leaf04(config-router-af)# exit-address-family
leaf04(config-router)# exit
leaf04(config)# router bgp 65104 vrf RED
leaf04(config-router)# bgp router-id 10.10.10.4
leaf04(config-router)# address-family ipv4 unicast
leaf04(config-router-af)# redistribute connected
leaf04(config-router-af)# exit-address-family
leaf04(config-router)# address-family l2vpn evpn
leaf04(config-router-af)# advertise ipv4 unicast
leaf04(config-router-af)# exit-address-family
leaf04(config-router)# exit
leaf04# write memory
leaf04# exit
cumulus@leaf04:~$
```

{{</tab>}}
{{<tab "spine01">}}

```
cumulus@spine01:~$ sudo vtysh
...
spine01# configure terminal
spine01(config)# interface lo
spine01(config-if)# ip pim
spine01(config-if)# ip pim use-source 10.10.10.101
spine01(config-if)# exit
spine01(config)# interface swp1
spine01(config-if)# ip pim
spine01(config-if)# exit
spine01(config)# interface swp2
spine01(config-if)# ip pim
spine01(config-if)# exit
spine01(config)# interface swp3
spine01(config-if)# ip pim
spine01(config-if)# exit
spine01(config)# interface swp4
spine01(config-if)# ip pim
spine01(config-if)# exit
spine01(config)# ip route 0.0.0.0/0 192.168.200.1 vrf mgmt
spine01(config)# ip pim rp 10.10.100.100 224.0.0.0/4
spine01(config)# ip pim ecmp
spine01(config)# ip pim keep-alive-timer 3600
spine01(config)# router bgp 65100
spine01(config-router)# bgp router-id 10.10.10.101
spine01(config-router)# bgp bestpath as-path multipath-relax
spine01(config-router)# neighbor underlay peer-group
spine01(config-router)# neighbor underlay remote-as external
spine01(config-router)# neighbor swp1 interface peer-group underlay
spine01(config-router)# neighbor swp2 interface peer-group underlay
spine01(config-router)# neighbor swp3 interface peer-group underlay
spine01(config-router)# neighbor swp4 interface peer-group underlay
spine01(config-router)# address-family ipv4 unicast
spine01(config-router-af)# redistribute connected
spine01(config-router-af)# neighbor underlay activate
spine01(config-router-af)# exit-address-family
spine01(config-router)# address-family l2vpn evpn
spine01(config-router-af)# neighbor underlay activate
spine01(config-router-af)# exit-address-family
spine01(config-router)# end
spine01# write memory
spine01# exit
cumulus@spine01:~$
```

{{</tab>}}
{{<tab "spine02">}}

```
cumulus@spine02:~$ sudo vtysh
...
spine02# configure terminal
spine02(config)# interface lo
spine02(config-if)# ip pim
spine02(config-if)# ip pim use-source 10.10.10.102
spine02(config-if)# exit
spine02(config)# interface swp1
spine02(config-if)# ip pim
spine02(config-if)# exit
spine02(config)# interface swp2
spine02(config-if)# ip pim
spine02(config-if)# exit
spine02(config)# interface swp3
spine02(config-if)# ip pim
spine02(config-if)# exit
spine02(config)# interface swp4
spine02(config-if)# ip pim
spine02(config-if)# exit
spine02(config)# ip route 0.0.0.0/0 192.168.200.1 vrf mgmt
spine02(config)# ip pim rp 10.10.100.100 224.0.0.0/4
spine02(config)# ip pim ecmp
spine02(config)# ip pim keep-alive-timer 3600
spine02(config)# router bgp 65100
spine02(config-router)# bgp router-id 10.10.10.102
spine02(config-router)# bgp bestpath as-path multipath-relax
spine02(config-router)# neighbor underlay peer-group
spine02(config-router)# neighbor underlay remote-as external
spine02(config-router)# neighbor swp1 interface peer-group underlay
spine02(config-router)# neighbor swp2 interface peer-group underlay
spine02(config-router)# neighbor swp3 interface peer-group underlay
spine02(config-router)# neighbor swp4 interface peer-group underlay
spine02(config-router)# address-family ipv4 unicast
spine02(config-router-af)# redistribute connected
spine02(config-router-af)# neighbor underlay activate
spine02(config-router-af)# exit-address-family
spine02(config-router)# address-family l2vpn evpn
spine02(config-router-af)# neighbor underlay activate
spine02(config-router-af)# exit-address-family
spine02(config-router)# end
spine02# write memory
spine02# exit
cumulus@spine02:~$
```

{{</tab>}}
{{</tabs>}}

{{</tab>}}
{{</tabs>}}

### /etc/network/interfaces

{{<tabs "/etc/network/interfaces">}}
{{<tab "leaf01">}}

```
cumulus@leaf01:~$ cat /etc/network/interfaces
...
auto lo
iface lo inet loopback
    address 10.10.10.1/32
    vxlan-local-tunnelip 10.10.10.1

auto eth0
iface eth0
    vrf mgmt
    address 192.168.200.11/24

auto mgmt
iface mgmt
  vrf-table auto
  address 127.0.0.1/8
  address ::1/128

auto RED
iface RED
  vrf-table auto

auto BLUE
iface BLUE
  vrf-table auto

auto bridge
iface bridge
    bridge-ports bond1 bond2 bond3 vni10 vni20 vni30 vniRED vniBLUE 
    bridge-vids 10 20 30 4001 4002  
    bridge-vlan-aware yes

auto vni10
iface vni10
    bridge-access 10
    vxlan-id 10
    mstpctl-portbpdufilter yes
    mstpctl-bpduguard yes
    bridge-learning off
    bridge-arp-nd-suppress on
    vxlan-mcastgrp 224.0.0.10

auto vni20
iface vni20
    bridge-access 20
    vxlan-id 20
    mstpctl-portbpdufilter yes
    mstpctl-bpduguard yes
    bridge-learning off
    bridge-arp-nd-suppress on
    vxlan-mcastgrp 224.0.0.20

auto vni30
iface vni30
    bridge-access 30
    vxlan-id 30
    mstpctl-portbpdufilter yes
    mstpctl-bpduguard yes
    bridge-learning off
    bridge-arp-nd-suppress on
    vxlan-mcastgrp 224.0.0.30

auto vniRED
iface vniRED
    bridge-access 4001
    vxlan-id 4001
    mstpctl-portbpdufilter yes
    mstpctl-bpduguard yes
    bridge-learning off
    bridge-arp-nd-suppress on

auto vniBLUE
iface vniBLUE
    bridge-access 4002
    vxlan-id 4002
    mstpctl-portbpdufilter yes
    mstpctl-bpduguard yes
    bridge-learning off
    bridge-arp-nd-suppress on

auto vlan10
iface vlan10
    address 10.1.10.2/24
    address-virtual 00:00:00:00:00:10 10.1.10.1/24
    vrf RED
    vlan-raw-device bridge
    vlan-id 10

auto vlan20
iface vlan20
    address 10.1.20.2/24
    address-virtual 00:00:00:00:00:20 10.1.20.1/24
    vrf RED
    vlan-raw-device bridge
    vlan-id 20

auto vlan30
iface vlan30
    address 10.1.30.2/24
    address-virtual 00:00:00:00:00:30 10.1.30.1/24
    vrf BLUE
    vlan-raw-device bridge
    vlan-id 30

auto vlan4001
iface vlan4001
    hwaddress 44:38:39:BE:EF:AA
    vrf RED
    vlan-raw-device bridge
    vlan-id 4001

auto vlan4002
iface vlan4002
    hwaddress 44:38:39:BE:EF:AA
    vrf BLUE
    vlan-raw-device bridge
    vlan-id 4002

auto swp51
iface swp51
    alias to spine

auto swp52
iface swp52
    alias to spine

auto swp1
iface swp1
    alias bond member of bond1

auto bond1
iface bond1
    bond-slaves swp1 
    es-sys-mac 44:38:39:BE:EF:AA
    bridge-access 10
    mtu 9000
    bond-lacp-bypass-allow yes
    mstpctl-bpduguard yes
    mstpctl-portadminedge yes

auto swp2
iface swp2
    alias bond member of bond2

auto bond2
iface bond2
    bond-slaves swp2 
    es-sys-mac 44:38:39:BE:EF:AA
    bridge-access 20
    mtu 9000
    bond-lacp-bypass-allow yes
    mstpctl-bpduguard yes
    mstpctl-portadminedge yes

auto swp3
iface swp3
    alias bond member of bond3

auto bond3
iface bond3
    bond-slaves swp3 
    es-sys-mac 44:38:39:BE:EF:AA
    bridge-access 30
    mtu 9000
    bond-lacp-bypass-allow yes
    mstpctl-bpduguard yes
    mstpctl-portadminedge yes
```

{{</tab>}}
{{<tab "leaf02">}}

```
cumulus@leaf02:~$ cat /etc/network/interfaces
...
auto lo
iface lo inet loopback
    address 10.10.10.2/32
    vxlan-local-tunnelip 10.10.10.2

auto eth0
iface eth0
    vrf mgmt
    address 192.168.200.12/24

auto mgmt
iface mgmt
  vrf-table auto
  address 127.0.0.1/8
  address ::1/128

auto RED
iface RED
  vrf-table auto

auto BLUE
iface BLUE
  vrf-table auto

auto bridge
iface bridge
    bridge-ports bond1 bond2 bond3 vni10 vni20 vni30 vniRED vniBLUE 
    bridge-vids 10 20 30 4001 4002  
    bridge-vlan-aware yes

auto vni10
iface vni10
    bridge-access 10
    vxlan-id 10
    mstpctl-portbpdufilter yes
    mstpctl-bpduguard yes
    bridge-learning off
    bridge-arp-nd-suppress on
    vxlan-mcastgrp 224.0.0.10

auto vni20
iface vni20
    bridge-access 20
    vxlan-id 20
    mstpctl-portbpdufilter yes
    mstpctl-bpduguard yes
    bridge-learning off
    bridge-arp-nd-suppress on
    vxlan-mcastgrp 224.0.0.20

auto vni30
iface vni30
    bridge-access 30
    vxlan-id 30
    mstpctl-portbpdufilter yes
    mstpctl-bpduguard yes
    bridge-learning off
    bridge-arp-nd-suppress on
    vxlan-mcastgrp 224.0.0.30

auto vniRED
iface vniRED
    bridge-access 4001
    vxlan-id 4001
    mstpctl-portbpdufilter yes
    mstpctl-bpduguard yes
    bridge-learning off
    bridge-arp-nd-suppress on

auto vniBLUE
iface vniBLUE
    bridge-access 4002
    vxlan-id 4002
    mstpctl-portbpdufilter yes
    mstpctl-bpduguard yes
    bridge-learning off
    bridge-arp-nd-suppress on

auto vlan10
iface vlan10
    address 10.1.10.3/24
    address-virtual 00:00:00:00:00:10 10.1.10.1/24
    vrf RED
    vlan-raw-device bridge
    vlan-id 10

auto vlan20
iface vlan20
    address 10.1.20.3/24
    address-virtual 00:00:00:00:00:20 10.1.20.1/24
    vrf RED
    vlan-raw-device bridge
    vlan-id 20

auto vlan30
iface vlan30
    address 10.1.30.3/24
    address-virtual 00:00:00:00:00:30 10.1.30.1/24
    vrf BLUE
    vlan-raw-device bridge
    vlan-id 30

auto vlan4001
iface vlan4001
    hwaddress 44:38:39:BE:EF:AA
    vrf RED
    vlan-raw-device bridge
    vlan-id 4001

auto vlan4002
iface vlan4002
    hwaddress 44:38:39:BE:EF:AA
    vrf BLUE
    vlan-raw-device bridge
    vlan-id 4002

auto swp51
iface swp51
    alias to spine

auto swp52
iface swp52
    alias to spine

auto swp1
iface swp1
    alias bond member of bond1

auto bond1
iface bond1
    bond-slaves swp1 
    es-sys-mac 44:38:39:BE:EF:AA
    bridge-access 10
    mtu 9000
    bond-lacp-bypass-allow yes
    mstpctl-bpduguard yes
    mstpctl-portadminedge yes

auto swp2
iface swp2
    alias bond member of bond2

auto bond2
iface bond2
    bond-slaves swp2 
    es-sys-mac 44:38:39:BE:EF:AA
    bridge-access 20
    mtu 9000
    bond-lacp-bypass-allow yes
    mstpctl-bpduguard yes
    mstpctl-portadminedge yes

auto swp3
iface swp3
    alias bond member of bond3

auto bond3
iface bond3
    bond-slaves swp3 
    es-sys-mac 44:38:39:BE:EF:AA
    bridge-access 30
    mtu 9000
    bond-lacp-bypass-allow yes
    mstpctl-bpduguard yes
    mstpctl-portadminedge yes
```

{{</tab>}}
{{<tab "leaf03">}}

```
cumulus@leaf03:~$ cat /etc/network/interfaces
...
auto lo
iface lo inet loopback
    address 10.10.10.3/32
    vxlan-local-tunnelip 10.10.10.3

auto eth0
iface eth0
    vrf mgmt
    address 192.168.200.13/24

auto mgmt
iface mgmt
  vrf-table auto
  address 127.0.0.1/8
  address ::1/128

auto RED
iface RED
  vrf-table auto

auto BLUE
iface BLUE
  vrf-table auto

auto bridge
iface bridge
    bridge-ports bond1 bond2 bond3 vni10 vni20 vni30 vniRED vniBLUE 
    bridge-vids 10 20 30 4001 4002  
    bridge-vlan-aware yes

auto vni10
iface vni10
    bridge-access 10
    vxlan-id 10
    mstpctl-portbpdufilter yes
    mstpctl-bpduguard yes
    bridge-learning off
    bridge-arp-nd-suppress on
    vxlan-mcastgrp 224.0.0.10

auto vni20
iface vni20
    bridge-access 20
    vxlan-id 20
    mstpctl-portbpdufilter yes
    mstpctl-bpduguard yes
    bridge-learning off
    bridge-arp-nd-suppress on
    vxlan-mcastgrp 224.0.0.20

auto vni30
iface vni30
    bridge-access 30
    vxlan-id 30
    mstpctl-portbpdufilter yes
    mstpctl-bpduguard yes
    bridge-learning off
    bridge-arp-nd-suppress on
    vxlan-mcastgrp 224.0.0.30

auto vniRED
iface vniRED
    bridge-access 4001
    vxlan-id 4001
    mstpctl-portbpdufilter yes
    mstpctl-bpduguard yes
    bridge-learning off
    bridge-arp-nd-suppress on

auto vniBLUE
iface vniBLUE
    bridge-access 4002
    vxlan-id 4002
    mstpctl-portbpdufilter yes
    mstpctl-bpduguard yes
    bridge-learning off
    bridge-arp-nd-suppress on

auto vlan10
iface vlan10
    address 10.1.10.4/24
    address-virtual 00:00:00:00:00:10 10.1.10.1/24
    vrf RED
    vlan-raw-device bridge
    vlan-id 10

auto vlan20
iface vlan20
    address 10.1.20.4/24
    address-virtual 00:00:00:00:00:20 10.1.20.1/24
    vrf RED
    vlan-raw-device bridge
    vlan-id 20

auto vlan30
iface vlan30
    address 10.1.30.4/24
    address-virtual 00:00:00:00:00:30 10.1.30.1/24
    vrf BLUE
    vlan-raw-device bridge
    vlan-id 30

auto vlan4001
iface vlan4001
    hwaddress 44:38:39:BE:EF:BB
    vrf RED
    vlan-raw-device bridge
    vlan-id 4001

auto vlan4002
iface vlan4002
    hwaddress 44:38:39:BE:EF:BB
    vrf BLUE
    vlan-raw-device bridge
    vlan-id 4002

auto swp51
iface swp51
    alias to spine

auto swp52
iface swp52
    alias to spine

auto swp1
iface swp1
    alias bond member of bond1

auto bond1
iface bond1
    bond-slaves swp1 
    es-sys-mac 44:38:39:BE:EF:BB
    bridge-access 10
    mtu 9000
    bond-lacp-bypass-allow yes
    mstpctl-bpduguard yes
    mstpctl-portadminedge yes

auto swp2
iface swp2
    alias bond member of bond2

auto bond2
iface bond2
    bond-slaves swp2 
    es-sys-mac 44:38:39:BE:EF:BB
    bridge-access 20
    mtu 9000
    bond-lacp-bypass-allow yes
    mstpctl-bpduguard yes
    mstpctl-portadminedge yes

auto swp3
iface swp3
    alias bond member of bond3

auto bond3
iface bond3
    bond-slaves swp3 
    es-sys-mac 44:38:39:BE:EF:BB
    bridge-access 30
    mtu 9000
    bond-lacp-bypass-allow yes
    mstpctl-bpduguard yes
    mstpctl-portadminedge yes
```

{{</tab>}}
{{<tab "leaf04">}}

```
cumulus@leaf04:~$ cat /etc/network/interfaces
...
auto lo
iface lo inet loopback
    address 10.10.10.4/32
    vxlan-local-tunnelip 10.10.10.4

auto eth0
iface eth0
    vrf mgmt
    address 192.168.200.14/24

auto mgmt
iface mgmt
  vrf-table auto
  address 127.0.0.1/8
  address ::1/128

auto RED
iface RED
  vrf-table auto

auto BLUE
iface BLUE
  vrf-table auto

auto bridge
iface bridge
    bridge-ports bond1 bond2 bond3 vni10 vni20 vni30 vniRED vniBLUE 10 20 30 4001 4002  
    bridge-vlan-aware yes

auto vni10
iface vni10
    bridge-access 10
    vxlan-id 10
    mstpctl-portbpdufilter yes
    mstpctl-bpduguard yes
    bridge-learning off
    bridge-arp-nd-suppress on
    vxlan-mcastgrp 224.0.0.10

auto vni20
iface vni20
    bridge-access 20
    vxlan-id 20
    mstpctl-portbpdufilter yes
    mstpctl-bpduguard yes
    bridge-learning off
    bridge-arp-nd-suppress on
    vxlan-mcastgrp 224.0.0.20

auto vni30
iface vni30
    bridge-access 30
    vxlan-id 30
    mstpctl-portbpdufilter yes
    mstpctl-bpduguard yes
    bridge-learning off
    bridge-arp-nd-suppress on
    vxlan-mcastgrp 224.0.0.30

auto vniRED
iface vniRED
    bridge-access 4001
    vxlan-id 4001
    mstpctl-portbpdufilter yes
    mstpctl-bpduguard yes
    bridge-learning off
    bridge-arp-nd-suppress on

auto vniBLUE
iface vniBLUE
    bridge-access 4002
    vxlan-id 4002
    mstpctl-portbpdufilter yes
    mstpctl-bpduguard yes
    bridge-learning off
    bridge-arp-nd-suppress on

auto vlan10
iface vlan10
    address 10.1.10.5/24
    address-virtual 00:00:00:00:00:10 10.1.10.1/24
    vrf RED
    vlan-raw-device bridge
    vlan-id 10

auto vlan20
iface vlan20
    address 10.1.20.5/24
    address-virtual 00:00:00:00:00:20 10.1.20.1/24
    vrf RED
    vlan-raw-device bridge
    vlan-id 20

auto vlan30
iface vlan30
    address 10.1.30.5/24
    address-virtual 00:00:00:00:00:30 10.1.30.1/24
    vrf BLUE
    vlan-raw-device bridge
    vlan-id 30

auto vlan4001
iface vlan4001
    hwaddress 44:38:39:BE:EF:BB
    vrf RED
    vlan-raw-device bridge
    vlan-id 4001

auto vlan4002
iface vlan4002
    hwaddress 44:38:39:BE:EF:BB
    vrf BLUE
    vlan-raw-device bridge
    vlan-id 4002

auto swp51
iface swp51
    alias to spine

auto swp52
iface swp52
    alias to spine

auto swp1
iface swp1
    alias bond member of bond1

auto bond1
iface bond1
    bond-slaves swp1 
    es-sys-mac 44:38:39:BE:EF:BB
    bridge-access 10
    mtu 9000
    bond-lacp-bypass-allow yes
    mstpctl-bpduguard yes
    mstpctl-portadminedge yes

auto swp2
iface swp2
    alias bond member of bond2

auto bond2
iface bond2
    bond-slaves swp2 
    es-sys-mac 44:38:39:BE:EF:BB
    bridge-access 20
    mtu 9000
    bond-lacp-bypass-allow yes
    mstpctl-bpduguard yes
    mstpctl-portadminedge yes

auto swp3
iface swp3
    alias bond member of bond3

auto bond3
iface bond3
    bond-slaves swp3 
    es-sys-mac 44:38:39:BE:EF:BB
    bridge-access 30
    mtu 9000
    bond-lacp-bypass-allow yes
    mstpctl-bpduguard yes
    mstpctl-portadminedge yes
```

{{</tab>}}
{{<tab "spine01">}}

```
cumulus@spine01:~$ cat /etc/network/interfaces
...
auto lo
iface lo inet loopback
    address 10.10.10.101/32

auto eth0
iface eth0
    address 192.168.200.21/24
    vrf mgmt

auto swp1
iface swp1
    alias to leaf

auto swp2
iface swp2
    alias to leaf

auto swp3
iface swp3
    alias to leaf

auto swp4
iface swp4
    alias to leaf

auto mgmt
iface mgmt
    address 127.0.0.1/8
    address ::1/128
    vrf-table auto
```

{{</tab>}}
{{<tab "spine02">}}

```
cumulus@spine02:~$ cat /etc/network/interfaces
...
auto lo
iface lo inet loopback
    address 10.10.10.102/32

auto eth0
iface eth0
    address 192.168.200.21/24
    vrf mgmt

auto swp1
iface swp1
    alias to leaf

auto swp2
iface swp2
    alias to leaf

auto swp3
iface swp3
    alias to leaf

auto swp4
iface swp4
    alias to leaf

auto mgmt
iface mgmt
    address 127.0.0.1/8
    address ::1/128
    vrf-table auto
```

{{</tab>}}
{{<tab "host01">}}

```
cumulus@host01:~$ cat /etc/network/interfaces
...
auto lo
iface lo inet loopback

# The OOB network interface
auto eth0
iface eth0 inet dhcp

# The data plane network interfaces
auto eth1
iface eth1 inet manual
  # Required for Vagrant
  post-up ip link set promisc on dev eth1

auto eth2
iface eth2 inet manual
  # Required for Vagrant
  post-up ip link set promisc on dev eth2

auto uplink
iface uplink inet static
  address 10.1.10.101
  netmask 255.255.255.0
  mtu 9000
  bond-slaves eth1 eth2
  bond-mode 802.3ad
  bond-miimon 100
  bond-lacp-rate 1
  bond-min-links 1
  bond-xmit-hash-policy layer3+4
  post-up ip route add 10.0.0.0/8 via 10.1.10.1
```

{{</tab>}}
{{<tab "host02">}}

```
cumulus@host02:~$ cat /etc/network/interfaces
...
auto lo
iface lo inet loopback

# The OOB network interface
auto eth0
iface eth0 inet dhcp

# The data plane network interfaces
auto eth1
iface eth1 inet manual
  # Required for Vagrant
  post-up ip link set promisc on dev eth1

auto eth2
iface eth2 inet manual
  # Required for Vagrant
  post-up ip link set promisc on dev eth2

auto uplink
iface uplink inet static
  address 10.1.20.102
  netmask 255.255.255.0
  mtu 9000
  bond-slaves eth1 eth2
  bond-mode 802.3ad
  bond-miimon 100
  bond-lacp-rate 1
  bond-min-links 1
  bond-xmit-hash-policy layer3+4
  post-up ip route add 10.0.0.0/8 via 10.1.20.1
```

{{</tab>}}
{{<tab "host03">}}

```
cumulus@host03:~$ cat /etc/network/interfaces
...
auto lo
iface lo inet loopback

# The OOB network interface
auto eth0
iface eth0 inet dhcp

# The data plane network interfaces
auto eth1
iface eth1 inet manual
  # Required for Vagrant
  post-up ip link set promisc on dev eth1

auto eth2
iface eth2 inet manual
  # Required for Vagrant
  post-up ip link set promisc on dev eth2

auto uplink
iface uplink inet static
  address 10.1.30.103
  netmask 255.255.255.0
  mtu 9000
  bond-slaves eth1 eth2
  bond-mode 802.3ad
  bond-miimon 100
  bond-lacp-rate 1
  bond-min-links 1
  bond-xmit-hash-policy layer3+4
  post-up ip route add 10.0.0.0/8 via 10.1.30.1
```

{{</tab>}}
{{<tab "host04">}}

```
cumulus@host04:~$ cat /etc/network/interfaces
...
auto lo
iface lo inet loopback

# The OOB network interface
auto eth0
iface eth0 inet dhcp

# The data plane network interfaces
auto eth1
iface eth1 inet manual
  # Required for Vagrant
  post-up ip link set promisc on dev eth1

auto eth2
iface eth2 inet manual
  # Required for Vagrant
  post-up ip link set promisc on dev eth2

auto uplink
iface uplink inet static
  address 10.1.10.104
  netmask 255.255.255.0
  mtu 9000
  bond-slaves eth1 eth2
  bond-mode 802.3ad
  bond-miimon 100
  bond-lacp-rate 1
  bond-min-links 1
  bond-xmit-hash-policy layer3+4
  post-up ip route add 10.0.0.0/8 via 10.1.10.1
```

{{</tab>}}
{{</tabs>}}

### /etc/frr/frr.conf

{{<tabs "frr.conf Files">}}
{{<tab "leaf01">}}

```
cumulus@leaf01:~$ cat /etc/frr/frr.conf
...
!
ip pim rp 10.10.100.100 224.0.0.0/4
ip pim ecmp
ip pim keep-alive-timer 3600
interface lo
  ip igmp
  ip pim
  ip pim use-source 10.10.10.1
interface swp51
  evpn mh uplink
  ip pim
interface swp52
  evpn mh uplink
  ip pim
interface bond1
  evpn mh es-df-pref 50000
  evpn mh es-id 1
  evpn mh es-sys-mac 44:38:39:BE:EF:AA
interface bond2
  evpn mh es-df-pref 50000
  evpn mh es-id 2
  evpn mh es-sys-mac 44:38:39:BE:EF:AA
interface bond3
  evpn mh es-df-pref 50000
  evpn mh es-id 3
  evpn mh es-sys-mac 44:38:39:BE:EF:AA

 evpn mh startup-delay 10
vrf mgmt
 ip route 0.0.0.0/0 192.168.200.1
 exit-vrf
!
vrf RED
 vni 4001
 exit-vrf
!
vrf BLUE
 vni 4002
 exit-vrf
!
!
router bgp 65101
 bgp router-id 10.10.10.1
 neighbor underlay peer-group
 neighbor underlay remote-as external
 neighbor swp51 interface peer-group underlay
 neighbor swp52 interface peer-group underlay
 neighbor swp53 interface peer-group underlay
 neighbor swp54 interface peer-group underlay
 !
 !
 address-family ipv4 unicast
  redistribute connected
 exit-address-family
 !
 address-family l2vpn evpn
  neighbor underlay activate
  advertise-all-vni
 exit-address-family
!
router bgp 65101 vrf RED
 bgp router-id 10.10.10.1
 !
 address-family ipv4 unicast
  redistribute connected
 exit-address-family
 !
 address-family l2vpn evpn
  advertise ipv4 unicast
 exit-address-family
!
router bgp 65101 vrf BLUE
 bgp router-id 10.10.10.1
 !
 address-family ipv4 unicast
  redistribute connected
 exit-address-family
 !
 address-family l2vpn evpn
  advertise ipv4 unicast
 exit-address-family
!
```

{{</tab>}}
{{<tab "leaf02">}}

```
cumulus@leaf02:~$ cat /etc/frr/frr.conf
...
!
ip pim rp 10.10.100.100 224.0.0.0/4
ip pim ecmp
ip pim keep-alive-timer 3600
interface lo
  ip igmp
  ip pim
  ip pim use-source 10.10.10.2
interface swp51
  evpn mh uplink
  ip pim
interface swp52
  evpn mh uplink
  ip pim
interface bond1
  evpn mh es-df-pref 1
  evpn mh es-id 1
  evpn mh es-sys-mac 44:38:39:BE:EF:AA
interface bond2
  evpn mh es-df-pref 1
  evpn mh es-id 2
  evpn mh es-sys-mac 44:38:39:BE:EF:AA
interface bond3
  evpn mh es-df-pref 1
  evpn mh es-id 3
  evpn mh es-sys-mac 44:38:39:BE:EF:AA

 evpn mh startup-delay 10
vrf mgmt
 ip route 0.0.0.0/0 192.168.200.1
 exit-vrf
!
vrf RED
 vni 4001
 exit-vrf
!
vrf BLUE
 vni 4002
 exit-vrf
!
!
router bgp 65102
 bgp router-id 10.10.10.2
 neighbor underlay peer-group
 neighbor underlay remote-as external
 neighbor swp51 interface peer-group underlay
 neighbor swp52 interface peer-group underlay
 neighbor swp53 interface peer-group underlay
 neighbor swp54 interface peer-group underlay
 !
 !
 address-family ipv4 unicast
  redistribute connected
 exit-address-family
 !
 address-family l2vpn evpn
  neighbor underlay activate
  advertise-all-vni
 exit-address-family
!
router bgp 65102 vrf RED
 bgp router-id 10.10.10.2
 !
 address-family ipv4 unicast
  redistribute connected
 exit-address-family
 !
 address-family l2vpn evpn
  advertise ipv4 unicast
 exit-address-family
!
router bgp 65102 vrf BLUE
 bgp router-id 10.10.10.2
 !
 address-family ipv4 unicast
  redistribute connected
 exit-address-family
 !
 address-family l2vpn evpn
  advertise ipv4 unicast
 exit-address-family
!
```

{{</tab>}}
{{<tab "leaf03">}}

```
cumulus@leaf03:~$ cat /etc/frr/frr.conf
...
!
ip pim rp 10.10.100.100 224.0.0.0/4
ip pim ecmp
ip pim keep-alive-timer 3600
interface lo
  ip igmp
  ip pim
  ip pim use-source 10.10.10.3
interface swp51
  evpn mh uplink
  ip pim
interface swp52
  evpn mh uplink
  ip pim
interface bond1
  evpn mh es-df-pref 50000
  evpn mh es-id 1
  evpn mh es-sys-mac 44:38:39:BE:EF:BB
interface bond2
  evpn mh es-df-pref 50000
  evpn mh es-id 2
  evpn mh es-sys-mac 44:38:39:BE:EF:BB
interface bond3
  evpn mh es-df-pref 50000
  evpn mh es-id 3
  evpn mh es-sys-mac 44:38:39:BE:EF:BB

 evpn mh startup-delay 10
vrf mgmt
 ip route 0.0.0.0/0 192.168.200.1
 exit-vrf
!
vrf RED
 vni 4001
 exit-vrf
!
vrf BLUE
 vni 4002
 exit-vrf
!
!
router bgp 65103
 bgp router-id 10.10.10.3
 neighbor underlay peer-group
 neighbor underlay remote-as external
 neighbor swp51 interface peer-group underlay
 neighbor swp52 interface peer-group underlay
 neighbor swp53 interface peer-group underlay
 neighbor swp54 interface peer-group underlay
 !
 !
 address-family ipv4 unicast
  redistribute connected
 exit-address-family
 !
 address-family l2vpn evpn
  neighbor underlay activate
  advertise-all-vni
 exit-address-family
!
router bgp 65103 vrf RED
 bgp router-id 10.10.10.3
 !
 address-family ipv4 unicast
  redistribute connected
 exit-address-family
 !
 address-family l2vpn evpn
  advertise ipv4 unicast
 exit-address-family
!
router bgp 65103 vrf BLUE
 bgp router-id 10.10.10.3
 !
 address-family ipv4 unicast
  redistribute connected
 exit-address-family
 !
 address-family l2vpn evpn
  advertise ipv4 unicast
 exit-address-family
!
!
line vty
!
```

{{</tab>}}
{{<tab "leaf04">}}

```
cumulus@leaf03:~$ cat /etc/frr/frr.conf
...
!
ip pim rp 10.10.100.100 224.0.0.0/4
ip pim ecmp
ip pim keep-alive-timer 3600
interface lo
  ip igmp
  ip pim
  ip pim use-source 10.10.10.4
interface swp51
  evpn mh uplink
  ip pim
interface swp52
  evpn mh uplink
  ip pim
interface bond1
  evpn mh es-df-pref 1
  evpn mh es-id 1
  evpn mh es-sys-mac 44:38:39:BE:EF:BB
interface bond2
  evpn mh es-df-pref 1
  evpn mh es-id 2
  evpn mh es-sys-mac 44:38:39:BE:EF:BB
interface bond3
  evpn mh es-df-pref 1
  evpn mh es-id 3
  evpn mh es-sys-mac 44:38:39:BE:EF:BB

 evpn mh startup-delay 10
vrf mgmt
 ip route 0.0.0.0/0 192.168.200.1
 exit-vrf
!
vrf RED
 vni 4001
 exit-vrf
!
vrf BLUE
 vni 4002
 exit-vrf
!
!
router bgp 65104
 bgp router-id 10.10.10.4
 neighbor underlay peer-group
 neighbor underlay remote-as external
 neighbor swp51 interface peer-group underlay
 neighbor swp52 interface peer-group underlay
 neighbor swp53 interface peer-group underlay
 neighbor swp54 interface peer-group underlay
 !
 !
 address-family ipv4 unicast
  redistribute connected
 exit-address-family
 !
 address-family l2vpn evpn
  neighbor underlay activate
  advertise-all-vni
 exit-address-family
!
router bgp 65104 vrf RED
 bgp router-id 10.10.10.4
 !
 address-family ipv4 unicast
  redistribute connected
 exit-address-family
 !
 address-family l2vpn evpn
  advertise ipv4 unicast
 exit-address-family
!
router bgp 65104 vrf BLUE
 bgp router-id 10.10.10.4
 !
 address-family ipv4 unicast
  redistribute connected
 exit-address-family
 !
 address-family l2vpn evpn
  advertise ipv4 unicast
 exit-address-family
!
!
line vty
!
```

{{</tab>}}
{{<tab "spine01">}}

```
cumulus@spine01:~$ cat /etc/frr/frr.conf
...
!
ip pim rp 10.10.100.100 224.0.0.0/4
ip pim ecmp
ip pim keep-alive-timer 3600
interface lo
  ip igmp
  ip pim
  ip pim use-source 10.10.10.101
interface swp1
  ip pim
interface swp2
  ip pim
interface swp3
  ip pim
interface swp4
  ip pim
vrf mgmt
 ip route 0.0.0.0/0 192.168.200.1
 exit-vrf
!
!
router bgp 65100
 bgp router-id 10.10.10.101
 neighbor underlay peer-group
 neighbor underlay remote-as external
 neighbor swp1 interface peer-group underlay
 neighbor swp2 interface peer-group underlay
 neighbor swp3 interface peer-group underlay
 neighbor swp4 interface peer-group underlay
 !
 !
 address-family ipv4 unicast
  redistribute connected
 exit-address-family
 !
 address-family l2vpn evpn
  neighbor underlay activate
 exit-address-family
!
!
line vty
!
```

{{</tab>}}
{{<tab "spine02">}}

```
cumulus@spine02:~$ cat /etc/frr/frr.conf
!
ip pim rp 10.10.100.100 224.0.0.0/4
ip pim ecmp
ip pim keep-alive-timer 3600
interface lo
  ip igmp
  ip pim
  ip pim use-source 10.10.10.102
interface swp1
  ip pim
interface swp2
  ip pim
interface swp3
  ip pim
interface swp4
  ip pim
vrf mgmt
 ip route 0.0.0.0/0 192.168.200.1
 exit-vrf
!
!
router bgp 65100
 bgp router-id 10.10.10.102
 neighbor underlay peer-group
 neighbor underlay remote-as external
 neighbor swp1 interface peer-group underlay
 neighbor swp2 interface peer-group underlay
 neighbor swp3 interface peer-group underlay
 neighbor swp4 interface peer-group underlay
 neighbor swp5 interface peer-group underlay
 neighbor swp6 interface peer-group underlay
 !
 !
 address-family ipv4 unicast
  redistribute connected
 exit-address-family
 !
 address-family l2vpn evpn
  neighbor underlay activate
 exit-address-family
!
!
line vty
!
```

{{</tab>}}
{{</tabs>}}
