---
title: VXLAN Devices
author: NVIDIA
weight: 605
toc: 3
---
Cumulus Linux supports both single and traditional VXLAN devices.

{{%notice note%}}
- Single VXLAN devices are supported in VLAN-aware bridge mode only.
- A combination of single and traditional VXLAN devices is not supported.
{{%/notice%}}

## Traditional VXLAN Device

With a traditional VXLAN device, each VNI is represented as a separate device (for example, vni10, vni20, vni30).
You can configure traditional VXLAN devices with NCLU or by manually editing the `/etc/network/interfaces` file.

The following example configuration:
- Creates three unique VXLAN devices (vni10, vni20, and vni30)
- Adds each VXLAN device (vni10, vni20, and vni30) to the bridge called `bridge`
- Configures the local tunnel IP address to be the loopback address of the switch

{{< tabs "TabID25 ">}}
{{< tab "NCLU Commands ">}}

```
cumulus@leaf01:~$ net add interface swp1 bridge access 10
cumulus@leaf01:~$ net add interface swp2 bridge access 20
cumulus@leaf01:~$ net add vxlan vni10 vxlan id 10
cumulus@leaf01:~$ net add vxlan vni20 vxlan id 20
cumulus@leaf01:~$ net add bridge bridge ports vni10,vni20
cumulus@leaf01:~$ net add bridge bridge vids 10,20
cumulus@leaf01:~$ net add vxlan vni10 bridge access 10
cumulus@leaf01:~$ net add vxlan vni20 bridge access 20
cumulus@leaf01:~$ net add loopback lo vxlan local-tunnelip 10.10.10.1
cumulus@leaf01:~$ net pending
cumulus@leaf01:~$ net commit
```

{{< /tab >}}
{{< tab "NVUE Commands ">}}

NVUE commands are not supported.

{{< /tab >}}
{{< tab "Linux Commands ">}}

Edit the `/etc/network/interfaces` file, then run the `ifreload -a` command.

```
auto lo
iface lo inet loopback
    address 10.10.10.1/32
    vxlan-local-tunnelip 10.10.10.1

auto mgmt
iface mgmt
    address 127.0.0.1/8
    vrf-table auto

auto swp1
iface swp1
    bridge-access 10

auto swp2
iface swp2
    bridge-access 20

auto vni10
iface vni10
    bridge-access 10
    mstpctl-bpduguard yes
    mstpctl-portbpdufilter yes
    vxlan-id 10

auto vni20
iface vni20
    bridge-access 20
    mstpctl-bpduguard yes
    mstpctl-portbpdufilter yes
    vxlan-id 20

auto bridge
iface bridge
    bridge-ports swp1 swp2 vni10 vni20
    bridge-vlan-aware yes
    bridge-vids 10 20
    bridge-pvid 1
```

```
cumulus@leaf01:~$ ifreload -a
```

{{< /tab >}}
{{< /tabs >}}

## Single VXLAN Device

With a single VXLAN device, a set of VNIs are included in a single device model. The single VXLAN device has a set of attributes that belong to the VXLAN construct. Individual VNIs are represented as a VLAN to VNI mapping and you can specify which VLANs map to the associated VNIs. The single VXLAN device is similar to the VLAN-aware bridge model, where the bridge contains a set of VLANs and VNIs.

Cumulus Linux creates a unique name for the single VXLAN device in the format `vxlan<id>`, where the ID is generated using the bridge name as the hash key.

{{%notice note%}}
Cumulus Linux supports multiple single VXLAN devices.
{{%/notice%}}

You can configure a single VXLAN device with NVUE or by manually editing the `/etc/network/interfaces` file.

The following example configuration:
- Creates a single VXLAN device (vxlan48)
- Maps VLAN 10 to VNI 10 and VLAN 20 to VNI 20
- Adds the VXLAN device to the default bridge `br_default`
- Sets the flooding multicast group for VNI 10 to 239.1.1.110 and the multicast group for VNI 20 to 239.1.1.120

{{< tabs "TabID106 ">}}
{{< tab "NCLU Commands ">}}

NCLU commands are not supported.

{{< /tab >}}
{{< tab "NVUE Commands ">}}

```
cumulus@leaf01:~$ nv set bridge domain br_default vlan 10 vni 10
cumulus@leaf01:~$ nv set bridge domain br_default vlan 20 vni 20
cumulus@leaf01:~$ nv set nve vxlan source address 10.10.10.1
cumulus@leaf01:~$ nv set bridge domain br_default vlan 10 vni 10 flooding multicast-group 239.1.1.110
cumulus@leaf01:~$ nv set bridge domain br_default vlan 20 vni 20 flooding multicast-group 239.1.1.120
cumulus@leaf01:~$ nv set interface swp1 bridge domain br_default access 10
cumulus@leaf01:~$ nv set interface swp2 bridge domain br_default access 20
cumulus@leaf01:~$ nv config apply
```

The `nv config save` command creates the following configuration snippet in the `/etc/nvue.d/startup.yaml` file:

```
cumulus@spine01:~$ sudo cat /etc/nvue.d/startup.yaml
- set:
    bridge:
      domain:
        br_default:
          vlan:
            '10':
              vni:
                '10':
                  flooding:
                    multicast-group: 239.1.1.110
                    enable: on
            '20':
              vni:
                '20':
                  flooding:
                    multicast-group: 239.1.1.120
                    enable: on
    nve:
      vxlan:
        enable: on
        source:
          address: 10.10.10.1
    interface:
      swp1:
        bridge:
          domain:
            br_default:
              access: 10
        type: swp
      swp2:
        bridge:
          domain:
            br_default:
              access: 20
        type: swp
```

{{< /tab >}}
{{< tab "Linux Commands ">}}

Edit the `/etc/network/interfaces` file then run the `ifreload -a` command.

```
cumulus@leaf01:~$ sudo nano /etc/network/interfaces
...
auto swp1
iface swp1
    bridge-access 10

auto swp2
iface swp2
    bridge-access 20

auto vxlan48
iface vxlan48
    vxlan-mcastgrp-map 10=239.1.1.110 20=239.1.1.120
    bridge-vlan-vni-map 10=10 20=20
    bridge-vids 10 20
    bridge-learning off

auto br_default
iface br_default
    bridge-ports swp1 swp2 vxlan48
    hwaddress 44:38:39:22:01:ab
    bridge-vlan-aware yes
    bridge-vids 10 20
    bridge-pvid 1
```

```
cumulus@leaf01:~$ ifreload -a
```

{{< /tab >}}
{{< /tabs >}}

- For information about VXLAN devices and static VXLAN tunnels, see {{<link url="Static-VXLAN-Tunnels" text="Static VXLAN Tunnels">}}.
- For information about VXLAN devices and EVPN, see {{<link url="Ethernet-Virtual-Private-Network-EVPN" text="EVPN">}}.