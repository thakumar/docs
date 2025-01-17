---
title: Policy-based Routing
author: NVIDIA
weight: 760
toc: 3
---
Typical routing systems and protocols forward traffic based on the destination address in the packet, which is used to look up an entry in a routing table. However, sometimes the traffic on your network requires a more hands-on approach. You might need to forward a packet based on the source address, the packet size, or other information in the packet header.

Policy-based routing (PBR) lets you make routing decisions based on filters that change the routing behavior of specific traffic so that you can override the routing table and influence where the traffic goes. For example, you can use PBR to help you reach the best bandwidth utilization for business-critical applications, isolate traffic for inspection or analysis, or manually load balance outbound traffic.

Policy-based routing is applied to incoming packets. All packets received on a PBR-enabled interface pass through enhanced packet filters that determine rules and specify where to forward the packets.

{{%notice note%}}
- You can create a *maximum* of 255 PBR match rules and 256 next hop groups (this is the ECMP limit).
- You can apply only one PBR policy per input interface.
- You can match on *source* and *destination* IP address, or match on Differentiated Services Code Point (DSCP) or Explicit Congestion Notification (ECN) values within a packet.
- PBR is not supported for GRE or VXLAN tunneling.
- PBR is not supported on management interfaces, such as eth0.
- A PBR rule cannot contain both IPv4 and IPv6 addresses.
{{%/notice%}}

## Configure PBR

A PBR policy contains one or more policy maps. Each policy map:

- Is identified with a unique map name and sequence (rule) number. The rule number is used to determine the relative order of the map within the policy.
- Contains a match source IP rule and (or) a match destination IP rule and a set rule, or a match DSCP or ECN rule and a set rule.
   - To match on a source and destination address, a policy map can contain both match source and match destination IP rules.
   - A set rule determines the PBR next hop for the policy. <!--The set rule can contain a single next hop IP address or it can contain a next hop group. A next hop group has more than one next hop IP address so that you can use multiple interfaces to forward traffic. To use ECMP, you configure a next hop group.-->

To use PBR in Cumulus linux, you define a PBR policy and apply it to the ingress interface (the interface must already have an IP address assigned). Traffic is matched against the match rules in sequential order and forwarded according to the set rule in the first match. Traffic that does not match any rule is passed onto the normal destination based routing mechanism.

To configure a PBR policy:

{{< tabs "TabID45 ">}}
{{< tab "CUE Commands ">}}

1. Configure the policy map.

    The example commands below configure a policy map called `map1` with rule number 1 that matches on destination address 10.1.2.0/24 and source address 10.1.4.1/24.

    If the IP address in the rule is `0.0.0.0/0 or ::/0`, any IP address is a match. You cannot mix IPv4 and IPv6 addresses in a rule.

    ```
    cumulus@switch:~$ cl set router pbr map map1 rule 1 match destination-ip 10.1.2.0/24
    cumulus@switch:~$ cl set router pbr map map1 rule 1 match source-ip 10.1.4.1/24 
    ```

    Instead of matching on IP address, you can match packets according to the DSCP or ECN field in the IP header. The DSCP value can be an integer between 0 and 63 or the DSCP codepoint name. The ECN value can be an integer between 0 and 3. The following example command configures a policy map called `map1` with rule number 1 that matches on packets with the DSCP value 10:

    ```
    cumulus@switch:~$ cl set router pbr map map1 rule 1 match dscp 10
    ```

    The following example command configures a policy map called `map1` with rule number 1 that matches on packets with the ECN value 2:

    ```
    cumulus@switch:~$ cl set router pbr map map1 rule 1 match ecn 2
    ```

2. Apply a next hop group to the policy map. First configure the next hop group, then apply the group to the policy map. The example commands below create a next hop group called `group1` that contains the next hop 192.168.0.21 on output interface swp1 and VRF `RED` and the next hop 192.168.0.22, then applies the next hop group `group1` to the `map1` policy map.

    The output interface and VRF are optional. However, you must specify the VRF if the next hop is not in the default VRF.

    ```
    cumulus@switch:~$ cl set router pbr nexthop-group group1 via 192.168.0.21 interface swp1
    cumulus@switch:~$ cl set router pbr nexthop-group group1 via 192.168.0.21 vrf RED
    cumulus@switch:~$ cl set router pbr nexthop-group group1 via 192.168.0.22
    cumulus@switch:~$ cl set router pbr map map1 rule 1 action nexthop-group group1
    ```

   If you want the rule to use a specific VRF table as its lookup, set the VRF. If no VRF is set, the rule uses the VRF table the interface is in as its lookup. The example command below sets the rule to use the `dmz` VRF table:

    ```
    cumulus@switch:~$ cl set router pbr map map1 rule 1 action vrf dmz
    ```

3. Assign the PBR policy to an ingress interface. The example command below assigns the PBR policy `map1` to interface swp51:

    ```
    cumulus@switch:~$ cl set interface swp51 router pbr map map1
    cumulus@switch:~$ cl config apply
    ```

{{< /tab >}}
{{< tab "vtysh Commands ">}}

1. Enable the `pbrd` service in the `/etc/frr/daemons` file:

    ```
    cumulus@leaf01:~$ sudo nano /etc/frr/daemons
    ...
    bgpd=yes
    ospfd=no
    ospf6d=no
    ripd=no
    ripngd=no
    isisd=no
    fabricd=no
    pimd=no
    ldpd=no
    nhrpd=no
    eigrpd=no
    babeld=no
    sharpd=no
    pbrd=yes
    ...
    ```

2. {{<cl/restart-frr>}}

3. Configure the policy map.

    The example commands below configure a policy map called `map1` with sequence number 1, that matches on destination address 10.1.2.0/24 and source address 10.1.4.1/24.

    ```
    cumulus@switch:~$ sudo vtysh

    switch# configure terminal
    switch(config)# pbr-map map1 seq 1
    switch(config-pbr-map)# match dst-ip 10.1.2.0/24
    switch(config-pbr-map)# match src-ip 10.1.4.1/24
    switch(config-pbr-map)# exit
    switch(config)# 
    ```

    If the IP address in the rule is `0.0.0.0/0 or ::/0`, any IP address is a match. You cannot mix IPv4 and IPv6 addresses in a rule.

    Instead of matching on IP address, you can match packets according to the DSCP or ECN field in the IP header. The DSCP value can be an integer between 0 and 63 or the DSCP codepoint name. The ECN value can be an integer between 0 and 3. The following example command configures a policy map called `map1` with sequence number 1 that matches on packets with the DSCP value 10:

    ```
    switch# configure terminal
    switch(config)# pbr-map map1 seq 1
    switch(config-pbr-map)# match dscp 10
    switch(config-pbr-map)# exit
    switch(config)# 
    ```

    The following example command configures a policy map called `map1` with sequence number 1 that matches on packets with the ECN value 2:

    ```
    switch# configure terminal
    switch(config)# pbr-map map1 seq 1
    switch(config-pbr-map)# match ecn 2
    switch(config-pbr-map)# exit
    switch(config)# 
    ```

4. Apply a *next hop* group to the policy map. First configure the next hop group, then apply the group to the policy map. The example commands below create a next hop group called `group1` that contains the next hop 192.168.0.21 on output interface swp1 and VRF `RED`, and the next hop 192.168.0.22, then applies the next hop group `group1` to the `map1` policy map.

    The output interface and VRF are optional. However, you must specify the VRF if the next hop is not in the default VRF.

    ```
    switch(config)# nexthop-group group1
    switch(config-nh-group)# nexthop 192.168.0.21 swp1 nexthop-vrf RED
    switch(config-nh-group)# nexthop 192.168.0.22
    switch(config-nh-group)# exit
    switch(config)# pbr-map map1 seq 1
    switch(config-pbr-map)# set nexthop-group group1
    switch(config-pbr-map)# exit
    switch(config)#
    ```

    If you want the rule to use a specific VRF table as its lookup, set the VRF. If no VRF is set, the rule uses the VRF table the interface is in as its lookup. The example command below sets the rule to use the `dmz` VRF table:

    ```
    switch(config)# pbr-map map1 seq 1
    switch(config-pbr-map)# set vrf dmz
    switch(config-pbr-map)# exit
    switch(config)#
    ```

   Instead of a next hop *group*, you can apply a next hop to the policy map. The example command below applies the next hop 192.168.0.31 on the output interface swp2 and VRF `RED` to the `map1` policy map. The next hop must be an IP address. The output interface and VRF are optional, however, you *must* specify the VRF you want to use for resolution if the next hop is *not* in the default VRF.

    ```
    switch(config-pbr-map)# set nexthop 192.168.0.31 swp2 nexthop-vrf RED
    switch(config-pbr-map)# exit
    switch(config)#
    ```

5. Assign the PBR policy to an ingress interface. The example command below assigns the PBR policy `map1` to interface swp51:

    ```
    switch(config)# interface swp51
    switch(config-if)# pbr-policy map1
    switch(config-if)# end
    switch# write memory
    switch# exit
    cumulus@switch:~$
    ```

The vtysh commands save the configuration in the `/etc/frr/frr.conf` file. For example:

```
...
nexthop-group group1
nexthop 192.168.0.21 nexthop-vrf RED swp1
nexthop 192.168.0.22
pbr-map map1 seq 1
match dst-ip 10.1.2.0/24
match src-ip 10.1.4.1/24
set nexthop-group group1
interface swp51
pbr-policy map1
...
```

{{< /tab >}}
{{< /tabs >}}

{{%notice note%}}
You can only set one policy per interface.
{{%/notice%}}

## Modify PBR Rules

When you want to change or extend an existing PBR rule, you must first delete the conditions in the rule, then add the rule back with the modification or addition.

{{< expand " Modify an existing match/set condition"  >}}

The example below shows an existing configuration.

```
cumulus@switch:~$ net show pbr map
Seq: 4 rule: 303 Installed: yes Reason: Valid
    SRC Match: 10.1.4.1/24
    DST Match: 10.1.2.0/24
 nexthop 192.168.0.21
    Installed: yes Tableid: 10009
```

The CUE commands for the above configuration are:

```
cumulus@switch:~$ cl set router pbr map pbr-policy rule 4 match source-ip 10.1.4.1/24
cumulus@switch:~$ cl set router pbr map pbr-policy rule 4 match destination-ip 10.1.2.0/24
cumulus@switch:~$ cl set router pbr nexthop-group group1 via 192.168.0.21
cumulus@switch:~$ cl set router pbr map pbr-policy rule 4 action nexthop-group group1
```

To change the source IP match from 10.1.4.**1**/24 to 10.1.4.**2**/24, you must delete the existing sequence by explicitly specifying the match/set condition. For example:

```
cumulus@switch:~$ cl unset router pbr map pbr-policy rule 4 match source-ip
cumulus@switch:~$ cl unset router pbr map pbr-policy rule 4 match destination-ip
cumulus@switch:~$ cl unset router pbr nexthop-group group1 via 192.168.0.21
```

Add the new rule with the following NCLU commands:

```
cumulus@switch:~$ cl set router pbr map pbr-policy rule 4 match source-ip 10.1.4.2/24
cumulus@switch:~$ cl set router pbr map pbr-policy rule 4 match destination-ip 10.1.2.0/24
cumulus@switch:~$ cl set router pbr nexthop-group group1 via 192.168.0.21
cumulus@switch:~$ cl config apply
```

Run the `net show pbr map` command to verify that the rule has the updated source IP match:

```
cumulus@switch:~$ net show pbr map
Seq: 4 rule: 303 Installed: yes Reason: Valid
     SRC Match: 10.1.4.2/24
     DST Match: 10.1.2.0/24
   nexthop 192.168.0.21
     Installed: yes Tableid: 10012
```

Run the `ip rule show` command to verify the entry in the kernel:

```
cumulus@switch:~$ ip rule show

303: from 10.1.4.1/24 to 10.1.4.2 iif swp16 lookup 10012
```

Run the following command to verify `switchd`:

```
cumulus@switch:~$ sudo cat /cumulus/switchd/run/iprule/show | grep 303 -A 1
303: from 10.1.4.1/24 to 10.1.4.2 iif swp16 lookup 10012
     [hwstatus: unit: 0, installed: yes, route-present: yes, resolved: yes, nh-valid: yes, nh-type: nh, ecmp/rif: 0x1, action: route,  hitcount: 0]
```

{{< /expand >}}

{{< expand "Add a match condition to an existing rule"  >}}

The example below shows an existing configuration, where only one source IP match is configured:

```
Seq: 3 rule: 302 Installed: yes Reason: Valid
	SRC Match: 10.1.4.1/24
nexthop 192.168.0.21
	Installed: yes Tableid: 10008
```

The CUE commands for the above configuration are:

```
cumulus@switch:~$ cl set router pbr map pbr-policy rule 3 match source-ip 10.1.4.1/24
cumulus@switch:~$ cl set router pbr nexthop-group group1 via 192.168.0.21
```

To add a destination IP match to the rule, you must delete the existing rule sequence:

```
cumulus@switch:~$ cl unset router pbr map pbr-policy rule 3 match source-ip
cumulus@switch:~$ cl unset router pbr nexthop-group group1 via 192.168.0.21
cumulus@switch:~$ cl config apply
```

Add back the source IP match and next hop condition, and add the new destination IP match (dst-ip 10.1.2.0/24):

```
cumulus@switch:~$ cl set router pbr map pbr-policy rule 3 match source-ip 10.1.4.1/24
cumulus@switch:~$ cl set router pbr map pbr-policy rule 3 match destination-ip 10.1.2.0/24
cumulus@switch:~$ cl set router pbr nexthop-group group1 via 192.168.0.21
cumulus@switch:~$ cl config apply
```

Run the `net show pbr map` command to verify the update:

```
Seq: 3 rule: 302 Installed: 1(9) Reason: Valid
    SRC Match: 10.1.4.1/24
    DST Match: 10.1.2.0/24
   nexthop 192.168.0.21
    Installed: 1(1) Tableid: 10013
```

Run the `ip rule show` command to verify the entry in the kernel:

```
302:   from 10.1.4.1/24 to 10.1.2.0 iif swp16 lookup 10013
```

Run the following command to verify `switchd`:

```
cumulus@mlx-2400-91:~$ cat /cumulus/switchd/run/iprule/show | grep 302 -A 1
302: from 10.1.4.1/24 to 10.1.2.0 iif swp16 lookup 10013
     [hwstatus: unit: 0, installed: yes, route-present: yes, resolved: yes, nh-valid: yes, nh-type: nh, ecmp/rif: 0x1, action: route,  hitcount: 0]
```

{{< /expand >}}

## Delete PBR Rules and Policies

You can delete a PBR rule, a next hop group, or a policy. The following commands provide examples.

{{%notice note%}}
Use caution when deleting PBR rules and next hop groups, as you might create an incorrect configuration for the PBR policy.
{{%/notice%}}

{{< tabs "TabID451 ">}}
{{< tab "CUE Commands ">}}

The following examples show how to delete a PBR rule match:

```
cumulus@switch:~$ cl unset router pbr map map1 rule 1 match destination-ip
cumulus@switch:~$ cl config apply
```

The following examples show how to delete a next hop from a group:

```
cumulus@switch:~$ cl unset router pbr nexthop-group group1 via 192.168.0.22
cumulus@switch:~$ cl config apply
```

The following examples show how to delete a next hop group:

```
cumulus@switch:~$ cl unset router pbr nexthop-group group1
cumulus@switch:~$ cl config apply
```

The following examples show how to delete a PBR policy so that the PBR interface is no longer receiving PBR traffic:

```
cumulus@switch:~$ cl unset interface swp51 router pbr map map1
cumulus@switch:~$ cl config apply
```

The following examples show how to delete a PBR rule:

```
cumulus@switch:~$ cl unset router pbr map map1
cumulus@switch:~$ cl config apply
```

{{< /tab >}}
{{< tab "vtysh Commands ">}}

The following examples show how to delete a PBR rule match:

```
cumulus@switch:~$ sudo vtysh
switch# configure terminal
switch(config)# pbr-map map1 seq 1
switch(config-pbr-map)# no match dst-ip 10.1.2.0/24
switch(config-pbr-map)# end
switch# write memory
switch# exit
cumulus@switch:~$
```

The following examples show how to delete a next hop from a group:

```
cumulus@switch:~$ sudo vtysh
switch# configure terminal
switch(config)# nexthop-group group1
switch(config-nh-group)# no nexthop 192.168.0.32 swp1 nexthop-vrf RED
switch(config-nh-group)# end
switch# write memory
switch# exit
cumulus@switch:~$
```

The following examples show how to delete a next hop group:

```
cumulus@switch:~$ sudo vtysh
switch# configure terminal
switch(config)# no nexthop-group group1
switch(config)# end
switch# write memory
switch# exit
cumulus@switch:~$
```

The following examples show how to delete a PBR policy so that the PBR interface is no longer receiving PBR traffic:

```
cumulus@switch:~$ sudo vtysh
switch# configure terminal
switch(config)# interface swp51
switch(config-if)# no pbr-policy map1
switch(config-if)# end
switch# write memory
switch# exit
cumulus@switch:~$
```

The following examples show how to delete a PBR rule:

```
cumulus@switch:~$ sudo vtysh
switch# configure terminal
switch(config)# no pbr-map map1 seq 1
switch(config)# end
switch# write memory
switch# exit
cumulus@switch:~$
```

{{< /tab >}}
{{< /tabs >}}

<!--NO LONGER TRUE? - If a PBR rule has multiple conditions (for example, a source IP match and a destination IP match), but you only want to delete one condition, you have to delete all conditions first, then re-add the ones you want to keep.

The example below shows an existing configuration that has a source IP match and a destination IP match.

```
Seq: 6 rule: 305 Installed: yes Reason: Valid
   SRC Match: 10.1.4.1/24
   DST Match: 10.1.2.0/24
nexthop 192.168.0.21
   Installed: yes Tableid: 10011
```

The CUE commands for the above configuration are:

```
cumulus@switch:~$ cl set router pbr map pbr-policy rule 6 match source-ip 10.1.4.1/24
cumulus@switch:~$ cl set router pbr map pbr-policy rule 6 match destination-ip 10.1.2.0/24
cumulus@switch:~$ cl set router pbr nexthop-group group1 via 192.168.0.21
```

To remove the destination IP match, you must first delete all existing conditions defined under this sequence:

```
cumulus@switch:~$ cl unset router pbr map pbr-policy rule 6 match source-ip 
cumulus@switch:~$ cl unset router pbr map pbr-policy rule 6 match destination-ip
cumulus@switch:~$ cl unset router pbr nexthop-group group1 via 192.168.0.21
cumulus@switch:~$ cl config apply
```

Then, add back the conditions you want to keep:

```
cumulus@switch:~$ cl set router pbr map pbr-policy rule 6 match source-ip 10.1.4.1/24
cumulus@switch:~$ cl unset router pbr nexthop-group group1 via 192.168.0.21
cumulus@switch:~$ cl config apply
```-->

## Troubleshooting

To see the policies applied to all interfaces on the switch, run the NCLU `net show pbr interface` command or the `vtysh` `show pbr interface` command. For example:

```
cumulus@switch:~$ net show pbr interface
swp51(53) with pbr-policy map1
```

To see the policies applied to a specific interface on the switch, add the interface name at the end of the command; for example, `net show pbr interface swp51` (or `show pbr interface swp51` in `vtysh`).

To see information about all policies, including mapped table and rule numbers, run the NCLU `net show pbr map` command or the vtysh `show pbr map` command. If the rule is not set, you see a reason why.

```
cumulus@switch:~$ net show pbr map
 pbr-map map1 valid: yes
  Seq: 700 rule: 999 Installed: yes Reason: Valid
      SRC Match: 10.0.0.1/32
  nexthop 192.168.0.32
      Installed: yes Tableid: 10003
  Seq: 701 rule: 1000 Installed: yes Reason: Valid
      SRC Match: 90.70.0.1/32
  nexthop 192.168.0.32
      Installed: yes Tableid: 10004
```

To see information about a specific policy, what it matches, and with which interface it is associated, add the map name at the end of the command; for example, `net show pbr map map1` (or `show pbr map map1` in `vtysh`).

To see information about all next hop groups, run the CUE `cl show router pbr nexthop-group` command or the vtysh `show pbr nexthop-group` command.

```
cumulus@switch:~$ show pbr nexthop-group
Nexthop-Group: map1701 Table: 10004 Valid: yes Installed: yes
Valid: yes nexthop 10.1.1.2
Nexthop-Group: map1700 Table: 10003 Valid: yes Installed: yes
Valid: yes nexthop 10.1.1.2
Nexthop-Group: group1 Table: 10000 Valid: yes Installed: yes
Valid: yes nexthop 192.168.10.0 bond1
Valid: yes nexthop 192.168.10.2
Valid: yes nexthop 192.168.10.3 vlan70
Nexthop-Group: group2 Table: 10001 Valid: yes Installed: yes
Valid: yes nexthop 192.168.8.1
Valid: yes nexthop 192.168.8.2
Valid: yes nexthop 192.168.8.3
```

To see information about a specific next hop group, add the group name at the end of the command; for example, `cl show router pbr nexthop-group group1` (or `show pbr nexthop-group group1` in vtysh).

{{%notice note%}}
A new Linux routing table ID is used for each next hop and next hop group.
{{%/notice%}}

## Example Configuration

In the following example, the PBR-enabled switch has a PBR policy to route all traffic from the Internet to a server that performs anti-DDOS. The traffic returns to the PBR-enabled switch after being cleaned and is then passed onto the regular destination based routing mechanism.

{{< img src = "/images/cumulus-linux/pbr-example.png" >}}

The configuration for the example above is:

{{< tabs "TabID197 ">}}
{{< tab "CUE Commands ">}}

```
cumulus@leaf01:~$ cl set router pbr map map1 rule 1 match source-ip 0.0.0.0/0
cumulus@leaf01:~$ cl set router pbr nexthop-group group1 via 192.168.0.32
cumulus@leaf01:~$ cl set router pbr map map1 rule 1 action nexthop-group group1
cumulus@leaf01:~$ cl set interface swp51 router pbr map map1
cumulus@leaf01:~$ cl config
```

{{< /tab >}}
{{< tab "vtysh Commands ">}}

```
cumulus@switch:~$ sudo vtysh

switch# configure terminal
switch(config)# pbr-map map1 seq 1
switch(config-pbr-map)# match src-ip 0.0.0.0/0
switch(config-pbr-map)# set nexthop 192.168.0.32
switch(config-pbr-map)# exit
switch(config)# interface swp51
switch(config-if)# pbr-policy map1
switch(config-if)# end
switch# write memory
switch# exit
cumulus@switch:~$
```

The vtysh commands save the configuration in the `/etc/frr/frr.conf` file. For example:

```
nexthop-group group1
nexthop 192.168.0.32
pbr-map map1 seq 1
match src-ip 0.0.0.0/0
set nexthop-group group1
interface swp51
pbr-policy map1
...
```

{{< /tab >}}
{{< /tabs >}}
