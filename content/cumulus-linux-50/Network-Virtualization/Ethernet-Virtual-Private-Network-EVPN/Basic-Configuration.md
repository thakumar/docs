---
title: Basic Configuration
author: NVIDIA
weight: 550
toc: 4
---
The following sections provide the basic configuration needed to use EVPN as the control plane for VXLAN in a BGP-EVPN-based layer 2 extension deployment. For layer 3 multi-tenancy configuration, see {{<link url="Inter-subnet-Routing" text="Inter-subnet Routing">}}. For addtional EVPN configuration, see {{<link url="EVPN-Enhancements" text="EVPN Enhancements">}}.

## Basic EVPN Configuration Commands

Basic configuration in a BGP-EVPN-based layer 2 extension deployment requires you to:
- Configure VXLAN interfaces
- Configure BGP
- Activate the EVPN address family and enable EVPN between BGP neighbors

{{%notice note%}}
For a non-VTEP device that is only participating in EVPN route exchange, such as a spine switch where the network deployment uses hop-by-hop eBGP or the switch is acting as an iBGP route reflector, configuring VXLAN interfaces is not required.
{{%/notice%}}

You can configure EVPN with CUE commands or with Linux and vtysh commands.

{{< tabs "TabID78 ">}}
{{< tab "CUE Commands ">}}

1. Configure VXLAN Interfaces. The following example creates a single VXLAN interface (vxlan0), maps VLAN 10 to vni10 and VLAN 20 to vni20, adds the VXLAN device to the default bridge `br_default`, and sets the VXLAN local tunnel IP address to 10.10.10.10.

   ```
   cumulus@leaf01:~$ cl set bridge domain br_default vlan 10 vni 10
   cumulus@leaf01:~$ cl set bridge domain br_default vlan 20 vni 20
   cumulus@leaf01:~$ cl set nve vxlan source address 10.10.10.10
   cumulus@leaf01:~$ cl config apply
   ```

   To create a traditional VXLAN device, where each VNI is represented as a separate device instead of  a set of VNIs in a single device model, see {{<link url="VXLAN-Devices" text="VXLAN-Devices">}}.

2. Configure BGP. The following example commands assign an ASN and router ID to leaf01 and spine01, specify the interfaces between the two BGP peers, and the prefixes to originate. For complete information on how to configure BGP, see {{<link url="Border-Gateway-Protocol-BGP" text="Border Gateway Protocol - BGP">}}.

   {{< tabs "TabID25 ">}}
{{< tab "leaf01 ">}}

```
cumulus@leaf01:~$ cl set router bgp autonomous-system 65101
cumulus@leaf01:~$ cl set router bgp router-id 10.10.10.1
cumulus@leaf01:~$ cl set vrf default router bgp peer swp51 remote-as external
cumulus@leaf01:~$ cl set vrf default router bgp address-family ipv4-unicast static-network 10.10.10.1/32
cumulus@leaf01:~$ cl config apply
```

{{< /tab >}}
{{< tab "spine01 ">}}

```
cumulus@spine01:~$ cl set router bgp autonomous-system 651000
cumulus@spine01:~$ cl set router bgp router-id 10.10.10.101
cumulus@spine01:~$ cl set vrf default router bgp peer swp1 remote-as external
cumulus@spine01:~$ cl set vrf default router bgp address-family ipv4-unicast static-network 10.10.10.101/32
cumulus@spine01:~$ cl config apply
```

{{< /tab >}}
{{< /tabs >}}

3. Activate the EVPN address family and enable EVPN between BGP neighbors. The following example commands enable EVPN between leaf01 and spine01:

   {{< tabs "TabID119 ">}}
{{< tab "leaf01 ">}}

```
cumulus@leaf01:~$ cl set evpn enable on
cumulus@leaf01:~$ cl set vrf default router bgp address-family l2vpn-evpn enable on
cumulus@leaf01:~$ cl set vrf default router bgp peer swp51 address-family l2vpn-evpn enable on
cumulus@leaf01:~$ cl config apply
```

The CUE commands create the following configuration snippet in the `/etc/cue.d/startup.yaml` file:

```
cumulus@leaf01:~$ sudo cat /etc/cue.d/startup.yaml
- set:
    interface:
      lo:
        ip:
          address:
            10.10.10.1/32: {}
        type: loopback
    bridge:
      domain:
        br_default:
          vlan:
            '10':
              vni:
                '10': {}
            '20':
              vni:
                '20': {}
    nve:
      vxlan:
        enable: on
        source:
          address: 10.10.10.10
    router:
      bgp:
        autonomous-system: 65101
        enable: on
        router-id: 10.10.10.1
    vrf:
      default:
        router:
          bgp:
            peer:
              swp51:
                remote-as: external
                type: unnumbered
            enable: on
            address-family:
              ipv4-unicast:
                static-network:
                  10.10.10.1/32: {}
                enable: on
```

{{< /tab >}}
{{< tab "spine01 ">}}

```
cumulus@spine01:~$ cl set evpn enable on
cumulus@spine01:~$ cl set vrf default router bgp address-family l2vpn-evpn enable on
cumulus@spine01:~$ cl set vrf default router bgp peer swp1 address-family l2vpn-evpn enable on
cumulus@spine01:~$ cl config apply
```

The CUE commands create the following configuration snippet in the `/etc/cue.d/startup.yaml` file:

```
cumulus@spine01:~$ sudo cat /etc/cue.d/startup.yaml
- set:
    interface:
      lo:
        ip:
          address:
            10.10.10.101/32: {}
        type: loopback
    bridge:
      domain:
        br_default:
          vlan:
            '10':
              vni:
                '10': {}
            '20':
              vni:
                '20': {}
    nve:
      vxlan:
        enable: on
        source:
          address: 10.10.10.101
    router:
      bgp:
        autonomous-system: 651000
        enable: on
        router-id: 10.10.10.101
    vrf:
      default:
        router:
          bgp:
            peer:
              swp1:
                remote-as: external
                type: unnumbered
            enable: on
            address-family:
              ipv4-unicast:
                static-network:
                  10.10.10.101/32: {}
                enable: on
```

{{< /tab >}}
{{< /tabs >}}

{{< /tab >}}
{{< tab "Linux and vtysh Commands ">}}

1. Configure VXLAN Interfaces. Edit the `/etc/network/interfaces` file to create the VXLAN interfaces, attach them to a bridge, map the VLANs to the VNIs, and set the VXLAN local tunnel IP address. The example below creates two VXLAN interfaces (vni10 and vni20), maps VLAN 10 to vni10 and VLAN 20 to vni20, and sets the VXLAN local tunnel IP address to 10.10.10.10.

   ```
   cumulus@leaf01:~$ sudo nano /etc/network/interfaces
   ...
   auto lo
   iface lo inet loopback
           address 10.10.10.1/32
           vxlan-local-tunnelip 10.10.10.10

   auto vni10
   iface vni10
           bridge-access 10
           bridge-learning off
           vxlan-id 10

   auto vni20
   iface vni20
           bridge-access 20
           bridge-learning off
           vxlan-id 20

   auto vlan10
   iface vlan10
       vlan-raw-device br_default
       vlan-id 10

   auto vlan20
   iface vlan20
       vlan-raw-device br_default
       vlan-id 20

   auto br_default
   iface br_default
           bridge-ports vni10 vni20
           bridge-vlan-aware yes
           bridge-vids 10 20
           bridge-pvid 1
   ```

2. Configure BGP with vtysh commands. The following example commands assign an ASN and router ID to leaf01 and spine01, specify the interfaces between the two BGP peers, and the prefixes to originate. For complete information on how to configure BGP, see {{<link url="Border-Gateway-Protocol-BGP" text="Border Gateway Protocol - BGP">}}.

   {{< tabs "TabID117 ">}}
{{< tab "leaf01 ">}}

```
cumulus@leaf01:~$ sudo vtysh
leaf01# configure terminal
leaf01(config)# router bgp 65101
leaf01(config-router)# bgp router-id 10.10.10.1
leaf01(config-router)# neighbor swp51 remote-as external
leaf01(config-router)# address-family ipv4
leaf01(config-router-af)# network 10.10.10.1/32
leaf01(config-router-af)# end
leaf01# write memory
leaf01# exit
cumulus@leaf01:~$
```

{{< /tab >}}
{{< tab "spine01 ">}}

```
cumulus@spine01:~$ sudo vtysh
spine01# configure terminal
spine01(config)# router bgp 651000
spine01(config-router)# bgp router-id 10.10.10.101
spine01(config-router)# neighbor swp1 remote-as external
spine01(config-router)# address-family ipv4
spine01(config-router-af)# network 10.10.10.101/32
spine01(config-router-af)# end
spine01# write memory
spine01# exit
cumulus@spine01:~$
```

{{< /tab >}}
{{< /tabs >}}

3. Activate the EVPN address family and enable EVPN between BGP neighbors. The following example commands enable EVPN between leaf01 and spine01. The commands automatically provision all locally configured VNIs to be advertised by the BGP control plane.

   {{< tabs "TabID194 ">}}
{{< tab "leaf01 ">}}

```
cumulus@leaf01:~$ sudo vtysh

leaf01# configure terminal
leaf01(config)# router bgp 65101
leaf01(config-router)# bgp router-id 10.10.10.1
leaf01(config-router)# neighbor swp51 interface remote-as external
leaf01(config-router)# address-family l2vpn evpn
leaf01(config-router-af)# neighbor swp51 activate
leaf01(config-router-af)# advertise-all-vni
leaf01(config-router-af)# end
leaf01)# write memory
leaf01)# exit
cumulus@leaf01:~$
```

The vtysh commands create the following configuration snippet in the `/etc/frr/frr.conf` file.

```
...
router bgp 65101
  bgp router-id 10.10.10.1
  neighbor swp51 interface remote-as external
  address-family l2vpn evpn
neighbor swp1 activate
  advertise-all-vni
...
```

{{< /tab >}}
{{< tab "spine01 ">}}

```
cumulus@spine01:~$ sudo vtysh

spine01# configure terminal
spine01(config)# router bgp 651000
spine01(config-router)# bgp router-id 10.10.10.101
spine01(config-router)# neighbor swp1 interface remote-as external
spine01(config-router)# address-family l2vpn evpn
spine01(config-router-af)# neighbor swp51 activate
spine01(config-router-af)# end
spine01)# write memory
spine01)# exit
cumulus@spine01:~$
```

{{%notice note%}}
The `advertise-all-vni` option is only needed on leaf switches that are VTEPs. EVPN routes received from a BGP peer are accepted, even without this explicit EVPN configuration. These routes are maintained in the global EVPN routing table. However, they only become effective (imported into the per-VNI routing table and appropriate entries installed in the kernel) when the VNI corresponding to the received route is locally known.
{{%/notice%}}

The vtysh commands create the following configuration snippet in the `/etc/frr/frr.conf` file:

```
...
router bgp 651000
  bgp router-id 10.10.10.101
  neighbor swp1 interface remote-as external
  address-family l2vpn evpn
neighbor swp1 activate
...
```

{{< /tab >}}
{{< /tabs >}}

{{< /tab >}}
{{< /tabs >}}

## EVPN and VXLAN Active-active Mode

For EVPN in VXLAN active-active mode, both switches in the MLAG pair establish EVPN peering with other EVPN speakers (for example, with spine switches if using hop-by-hop eBGP) and inform about their locally known VNIs and MACs. When MLAG is active, both switches announce this information with the shared anycast IP address.

For active-active configuration, make sure that:

- The `clagd-vxlan-anycast-ip` and `vxlan-local-tunnelip` parameters are under the loopback stanza on both peers.
- The anycast address is advertised to the routed fabric from both peers.
- The VNIs are configured identically on both peers.
- The peerlink must belong to the bridge.

MLAG synchronizes information between the two switches in the MLAG pair; EVPN does not synchronize.

For type-5 routes in an EVPN *symmetric* configuration with VXLAN active-active mode, Cumulus Linux uses Primary IP Address Advertisement. For information on configuring Primary IP Address Advertisement, see {{<link url="Inter-subnet-Routing#advertise-primary-ip-address-vxlan-active-active-mode" text="Advertise Primary IP Address">}}.

For information about active-active VTEPs and anycast IP behavior, and for failure scenarios, see {{<link url="VXLAN-Active-active-Mode">}}.

## Considerations

- When EVPN is enabled on a VTEP, all locally defined VNIs on that switch and other information (such as MAC addresses) are advertised to EVPN peers. There is no provision to only announce certain VNIs.
- ND suppression only works with the Spectrum-A1 chip.
- ARP suppression is enabled by default in Cumulus Linux. However, in a {{<link url="VXLAN-Active-active-Mode" text="VXLAN active-active">}} configuration, ARPs are sometimes *not* suppressed. This is because the neighbor entries are not synchronized between the two switches operating in active-active mode by a control plane. This has no impact on forwarding.
- You must configure the overlay (tenants) in a specific VRF and separate from the underlay, which resides in the default VRF. Layer 3 VNI mapping for the default VRF is not supported.
- EVPN is not supported when {{<link title="Redistribute Neighbor" >}} is also configured. Enabling both features simultaneously causes instability in IPv4 and IPv6 neighbor entries.
- To conform to {{<exlink url="https://tools.ietf.org/html/rfc6514#section-5" text="RFC 6514">}}, Cumulus Linux implements a stricter check on a received type-3 route to ensure that it has the PMSI attribute with the replication type set to *ingress-replication*.
