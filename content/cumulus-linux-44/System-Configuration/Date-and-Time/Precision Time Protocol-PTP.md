---
title: Precision Time Protocol - PTP
author: NVIDIA
weight: 128
toc: 3
---
Cumulus Linux supports IEEE 1588-2008 Precision Timing Protocol (PTPv2), which defines the algorithm and method for synchronizing clocks of various devices across packet-based networks, including Ethernet switches and IP routers.

PTP is capable of sub-microsecond accuracy. The clocks are organized in a master-slave hierarchy, where the slaves are synchronized to their masters, which can be slaves to their own masters. The hierarchy is created and updated automatically by the best master clock (BMC) algorithm, which runs on every clock. The grandmaster clock is the top-level master and is typically synchronized using a Global Positioning System (GPS) time source to provide a high-degree of accuracy.

In the following example:
- Boundary clock 2 receives time from Master 1 (the grandmaster) on a PTP slave port, sets its clock and passes the time down from the PTP master port to Boundary clock 1. 
- Boundary clock 1 receives the time on a PTP slave port, sets its clock and passes the time down the hierarchy through the PTP master ports to the hosts that receive the time.

{{< img src = "/images/cumulus-linux/date-time-ptp-example.png" >}}

## Cumulus Linux and PTP

PTP in Cumulus Linux uses the `linuxptp` package that includes the following programs:
- `ptp4l` provides the PTP protocol and state machines
- `phc2sys` provides PTP Hardware Clock and System Clock synchronization
- `timemaster` provides System Clock and PTP synchronization
- `monitor` provides monitoring

{{%notice note%}}
- PTP is supported on Spectrum-2 and above.
- You cannot run both PTP and NTP on the switch.
- PTP is supported in boundary clock mode only (the switch provides timing to downstream servers; it is a slave to a higher-level clock and a master to downstream clocks).
- The switch uses hardware time stamping to capture timestamps from an Ethernet frame at the physical layer. This allows PTP to account for delays in message transfer and greatly improves the accuracy of time synchronization.
- Both IPv4 and IPv6 UDP PTP packets are supported.
- Only a single PTP domain per network is supported.
- PTP is supported on layer 3 interfaces, trunk ports, and switch ports belonging to a VLAN. PTP is not supported on bonds.
- You can isolate PTP traffic to a non-default VRF.
- Multicast and mixed message mode is supported; unicast only message mode is *not* supported.
{{%/notice%}}

## Basic Configuration

Basic PTP configuration requires you:

- Enable PTP on the switch to start the `ptp4l` and `phc2sys` processes.
- Configure the interfaces on the switch that you want to use for PTP. Each interface must be a layer 3 routed interface with an IP address. You do not need to specify which is a master interface and which is a slave interface; this is determined by the PTP packet received.

The basic configuration shown below uses the *default* PTP settings:
- Clock mode is set to Boundary. This is the only clock mode currently supported.
- PTP profile is set to default-1588; the profile specified in the IEEE 1588 standard. This is the only profile currently supported.
- {{<link url="#transport-mode" text="Transport mode">}} is set to IPv4.
- {{<link url="#ptp-clock-domain" text="PTP clock domain">}} is set to 0.
- {{<link url="#ptp-priority" text="PTP Priority1 and Priority2">}} are set to 128.
- {{<link url="#one-step-and-two-step-mode" text="Hardware packet time stamping mode" >}} is set to one-step.
- {{<link url="#diffserv-code-point-dscp" text="DSCP" >}} is set to 43 for both general and event messages.
- {{<link url="#acceptable-master-table" text="Announce messages from any master are accepted">}}.
- {{<link url="#message-mode" text="Message Mode">}} is multicast.

To configure optional settings, such as the PTP domain, priority, transport mode, DSCP, and timers, see {{<link url="#optional-configuration" text="Optional Configuration">}} below.

{{%notice note%}}
You can configure PTP with NVUE or by manually editing `/etc/cumulus/switchd.conf` file; NCLU configuration is not supported.
{{%/notice%}}

{{< tabs "TabID36 ">}}
{{< tab "NVUE Commands ">}}

The NVUE `nv set service PTP` commands require an instance number (1 in the example command below). This number is used for management purposes.

```
cumulus@switch:~$ nv set service ptp 1 enable on
cumulus@switch:~$ nv set interface swp1 ip address 10.0.0.9/32
cumulus@switch:~$ nv set interface swp2 ip address 10.0.0.10/32
cumulus@switch:~$ nv set interface swp1 ptp enable on
cumulus@switch:~$ nv set interface swp2 ptp enable on
cumulus@switch:~$ nv config apply
```

The configuration is saved in the `/etc/ptp4l.conf` file.

{{< /tab >}}
{{< tab "Linux Commands ">}}

1. Open the `/etc/cumulus/switchd.conf` file in a text editor and add the following line:

    ```
    ptp.timestamping = TRUE
    ```

2. Restart `switchd`:

    {{<cl/restart-switchd>}}

3. Edit the `Default interface options` section of the `/etc/ptp4l.conf` file to configure the interfaces on the switch that you want to use for PTP.

```
cumulus@switch:~$ sudo nano /etc/ptp4l.conf
...
[global]
#
# Default Data Set
#
slaveOnly               0
priority1               128
priority2               128
domainNumber            0

twoStepFlag             0
dscp_event              43
dscp_general            43

offset_from_master_min_threshold   -200
offset_from_master_max_threshold   200
mean_path_delay_threshold          1

#
# Run time options
#
logging_level           6
path_trace_enabled      0
use_syslog              1
verbose                 0
summary_interval        0

#
# Default interface options
#
time_stamping           software

# Interfaces in which ptp should be enabled
# these interfaces should be routed ports
# if an interface does not have an ip address
# the ptp4l will not work as expected.

[swp1]
logAnnounceInterval     0
logSyncInterval         -3
logMinDelayReqInterval  -3
announceReceiptTimeout  3
udp_ttl                 1
masterOnly              0
delay_mechanism         E2E
network_transport       UDPv4

[swp2]
logAnnounceInterval     0
logSyncInterval         -3
logMinDelayReqInterval  -3
announceReceiptTimeout  3
udp_ttl                 1
masterOnly              0
delay_mechanism         E2E
network_transport       UDPv4
...
```

4. Restart the `ptp4l` and `phc2sys` daemons:

    ```
    cumulus@switch:~$ sudo systemctl restart ptp4l.service phc2sys.service
    ```

5. Enable the services to start at boot time:

    ```
    cumulus@switch:~$ sudo systemctl enable ptp4l.service phc2sys.service
    ```

{{< /tab >}}
{{< /tabs >}}

## Optional Configuration

<!--### PTP Profiles

PTP profiles are a standardized set of configurations and rules intended to meet the requirements of a specific application. Profiles define required, allowed, and restricted PTP options, network restrictions, and performance requirements.

Cumulus Linux supports the following profiles:
- *Default* is the profile specified in the IEEE 1588 standard. If you do not choose a profile or perform any optional configuration, the PTP software is initialized with default values in the standard. The default profile addresses some common applications, such as Industrial Automation. It does not have any network restrictions and is used as the first profile to be tested in qualification of equipment.
- *AES67* is a standard developed by the Audio Engineering Society to support Audio Over IP and Audio Over Ethernet. The standard uses IPV4 multicast and IGMP. DiffServ and DSCP are used for setting priorities. This PTP profile allows you to combine audio streams at the receiver end and allows synchronization of multiple streams.
- *SMPTE ST-2059-2* is a standard developed by the Society of Motion Pictures and Television Engineers. The standard was developed specifically to synchronize video equipment in a professional broadcast environment. Strict timing is required to switch between video streams at the frame level in the nano second range. For example, an NFL broadcast typically has multiple cameras sending video streams . The clocks in these cameras need to be synchronized to the nano second level. This allows switching from one camera angle to another by switching at the frame level without viewers noticing a blank frame.

AES67 and SMPTE ST-2059-2 are PTP Multimedia profiles. These profiles support video and audio applications used in professional broadcast environments and synchronization of multiple video and audio streams across multiple devices.

The following table shows the default settings for each profile.

| Profile | Domain| Priority1| Priority2 | Announce | Announce Timeout| Sync     | Delay Request |
| ------- | ------| -------- | --------- | -------- | --------------- | -------- | ------------- |
| Default | Default: 0<br>Range: 0 to 127   | Default: 128<br>Range: 0 to 255 | Default: 128<br>Range: 0 to 255 | Default: 1 (1 per 2 s)<br>Range: 0 to 4 | Default: 3<br>Range: 2 to 10 | Default:0 (1/s)<br>Range: -1 to 1 | Default:0<br>Range: 0 to 5 |
| AES67   | Default: 0<br>Range: 0 to 127   | Default: 128<br>Range: 0 to 255 | Default: 128<br>Range: 0 to 255 | Default: 0 (1/s)<br>Range: 0 to 4 | Default: 3<br>Range: 2 to 10 | Default:-2 250 ms (4/s)<br>Range: -4 to 1 | Default: -2 250ms (4/s)<br>Range: -4 to 1 |
| SMPTE   | Default: 127<br>Range: 0 to 127 | Default: 128<br>Range: 0 to 255 | Default: 128<br>Range: 0 to 255 | Default: -2 250ms (4/s)<br>Range: -3 to 1 |Default: 3<br>Range: 2 to 10 | Default: -3 125ms (8/s)<br>Range: -7 to -1 |Default: -3 125ms (8/s)<br>Range: -7 to -1 |

To configure the switch to use the AES67 profile:

```
cumulus@switch:~$ nv set service ptp 1 profile-type aes67
cumulus@switch:~$ nv config apply
```

To configure the switch to use the SMPTE ST-2059-2 profile:

```
cumulus@switch:~$ nv set service ptp 1 profile-type smpte
cumulus@switch:~$ nv config apply
```

To set the profile back to the default:

```
cumulus@switch:~$ nv set service ptp 1 profile-type default-1588
cumulus@switch:~$ nv config apply
```
-->
### Clock Domains

PTP domains allow different independent timing systems to be present in the same network without confusing each other. A PTP domain is a network or a portion of a network within which all the clocks are synchronized. Every PTP message contains a domain number. A PTP instance is configured to work in only one domain and ignores messages that contain a different domain number.

You can specify multiple PTP clock domains. Each domain is completely isolated from other domains so that it is seen as a different PTP network. You can specify a number between 0 and 127.

The following example commands configure domain 3:

{{< tabs "TabID173 ">}}
{{< tab "NVUE Commands ">}}

```
cumulus@switch:~$ nv set service ptp 1 domain 3
cumulus@switch:~$ nv config apply
```

{{< /tab >}}
{{< tab "Linux Commands ">}}

Edit the `Default Data Set` section of the `/etc/ptp4l.conf` file to change the `domainNumber` setting, then restart the `ptp4l` and `phc2sys` daemons.

```
cumulus@switch:~$ sudo nano /etc/ptp4l.conf
[global]
#
# Default Data Set
#
slaveOnly               0
priority1               254
priority2               254
domainNumber            3
...
```

```
cumulus@switch:~$ sudo systemctl restart ptp4l.service phc2sys.service
```

{{< /tab >}}
{{< /tabs >}}

### PTP Priority

Use the PTP priority to select the best master clock. You can set priority 1 and 2:
- Priority 1 overrides the clock class and quality selection criteria to select the best master clock.
- Priority 2 is used to identify primary and backup clocks among identical redundant Grandmasters.

The range for both priority1 and priority2 is between 0 and 255. The default priority is 128. For the boundary clock, use a number above 128. The lower priority is applied first.

The following example commands set priority 1 and priority 2 to 200:

{{< tabs "TabID212 ">}}
{{< tab "NVUE Commands ">}}

```
cumulus@switch:~$ nv set service ptp 1 priority1 200
cumulus@switch:~$ nv set service ptp 1 priority2 200
cumulus@switch:~$ nv config apply
```

{{< /tab >}}
{{< tab "Linux Commands ">}}

Edit the `Default Data Set` section of the `/etc/ptp4l.conf` file to change the `priority1` and, or `priority2` setting, then restart the `ptp4l` and `phc2sys` daemons.

```
cumulus@switch:~$ sudo nano /etc/ptp4l.conf
[global]
#
# Default Data Set
#
slaveOnly               0
priority1               254
priority2               254
domainNumber            3
...
```

```
cumulus@switch:~$ sudo systemctl restart ptp4l.service phc2sys.service
```

{{< /tab >}}
{{< /tabs >}}

### One-step and Two-step Mode

The Cumulus Linux switch supports hardware packet time stamping and provides two modes:
- In *one-step* mode, the PTP packet is time stamped as it egresses the port and there is no need for a follow-up packet.
- In *two-step* mode, the time is noted when the PTP packet egresses the port and is sent in a separate (follow-up) message.

One-step mode is the default configuration. To configure the switch to use two-step mode:

{{< tabs "TabID272 ">}}
{{< tab "NVUE Commands ">}}

```
cumulus@switch:~$ nv set service ptp 1 two-step on
cumulus@switch:~$ nv config apply
```

{{< /tab >}}
{{< tab "Linux Commands ">}}

Edit the `Default Data Set` section of the `/etc/ptp4l.conf` file to change the `twoStepFlag` setting to 1, then restart the `ptp4l` and `phc2sys` daemons.

```
cumulus@switch:~$ sudo nano /etc/ptp4l.conf
[global]
#
# Default Data Set
#
slaveOnly               0
priority1               254
priority2               254
domainNumber            3

twoStepFlag             1
dscp_event              43
dscp_general            43
...
```

```
cumulus@switch:~$ sudo systemctl restart ptp4l.service phc2sys.service
```

{{< /tab >}}
{{< /tabs >}}

### DSCP

You can configure the DiffServ code point (DSCP) value for all PTP IPv4 packets originated locally. You can set a value between 0 and 63.

{{< tabs "TabID320 ">}}
{{< tab "NVUE Commands ">}}

```
cumulus@switch:~$ nv set service ptp 1 ip-dscp 22
cumulus@switch:~$ nv config apply
```

{{< /tab >}}
{{< tab "Linux Commands ">}}

Edit the `Default Data Set` section of the `/etc/ptp4l.conf` file to change the `dscp_event` setting for PTP messages that trigger a Time Stamp read from the clock and the `dscp_general` setting for PTP messages that carry commands, responses, information, or time stamps.

After you save the `/etc/ptp4l.conf` file, restart the `ptp4l` and `phc2sys` daemons.

```
cumulus@switch:~$ sudo nano /etc/ptp4l.conf
[global]
#
# Default Data Set
#
slaveOnly               0
priority1               254
priority2               254
domainNumber            3

twoStepFlag             1
dscp_event              22
dscp_general            22
...
```

```
cumulus@switch:~$ sudo systemctl restart ptp4l.service phc2sys.service
```

{{< /tab >}}
{{< /tabs >}}

### Transport Mode

By default, PTP messages are encapsulated in UDP/IPV4 frames. To configure PTP messages on an interface to be encapsulated in UDP/IPV6 frames:

{{< tabs "TabID274 ">}}
{{< tab "NVUE Commands ">}}

```
cumulus@switch:~$ nv set interface swp1 ptp transport ipv6
cumulus@switch:~$ nv config apply
```

{{< /tab >}}
{{< tab "Linux Commands ">}}

Edit the `Default interface options` section of the `/etc/ptp4l.conf` file to change the `network_transport` setting for the interface, then restart the `ptp4l` and `phc2sys` daemons.

```
cumulus@switch:~$ sudo nano /etc/ptp4l.conf
...
# Default interface options
#
time_stamping           software

# Interfaces in which ptp should be enabled
# these interfaces should be routed ports
# if an interface does not have an ip address
# the ptp4l will not work as expected.

[swp1]
logAnnounceInterval     0
logSyncInterval         -3
logMinDelayReqInterval  -3
announceReceiptTimeout  3
udp_ttl                 1
masterOnly              0
delay_mechanism         E2E
network_transport       UDPv6

[swp2]
logAnnounceInterval     0
logSyncInterval         -3
logMinDelayReqInterval  -3
announceReceiptTimeout  3
udp_ttl                 1
masterOnly              0
delay_mechanism         E2E
network_transport       UDPv6
...
```

```
cumulus@switch:~$ sudo systemctl restart ptp4l.service phc2sys.service
```

{{< /tab >}}
{{< /tabs >}}

### Forced Master

By default, ports configured for PTP are in auto mode, where the state of the port is determined by the BMC algorithm.

You can configure *Forced Master* mode on a PTP port so that it is always in a master state and the BMC algorithm is not run for this port. Any Announce messages received on this port are ignored.

{{< tabs "TabID384 ">}}
{{< tab "NVUE Commands ">}}

```
cumulus@switch:~$ nv set interface swp1 ptp forced-master on
cumulus@switch:~$ nv config apply
```

{{< /tab >}}
{{< tab "Linux Commands ">}}

Edit the `Default interface options` section of the `/etc/ptp4l.conf` file to change the `masterOnly` setting for the interface, then restart the `ptp4l` and `phc2sys` daemons.

```
cumulus@switch:~$ sudo nano /etc/ptp4l.conf
...
# Default interface options
#
time_stamping           software

# Interfaces in which ptp should be enabled
# these interfaces should be routed ports
# if an interface does not have an ip address
# the ptp4l will not work as expected.

[swp1]
logAnnounceInterval     0
logSyncInterval         -3
logMinDelayReqInterval  -3
announceReceiptTimeout  3
udp_ttl                 1
masterOnly              1
delay_mechanism         E2E
network_transport       UDPv4
...
```

```
cumulus@switch:~$ sudo systemctl restart ptp4l.service phc2sys.service
```

{{< /tab >}}
{{< /tabs >}}

### Message Mode

Cumulus Linux currently supports the following PTP message modes:
- *Multicast*, where the ports subscribe to two multicast addresses, one for event messages that are timestamped and the other for general messages that are not timestamped. The Sync message sent by the master is a multicast message and is received by all slave ports. This is required because the slaves need the master's time. The slave ports in turn generate a Delay Request to the master. This is a multicast message and is received not only by the master for which the message is intended, but also by other slave ports. Similarly, the master's Delay Response is also received by all slave ports in addition to the intended slave port. The slave ports receiving the unintended Delay Requests and Responses need to drop the packets. This can affect network bandwidth, especially if there are hundreds of slave ports.
- *Mixed*, where Sync and Announce messages are sent as multicast messages but Delay Request and Response messages are sent as unicast. This avoids the issue seen in multicast message mode where every slave port sees Delay Requests and Responses from every other slave port.

Multicast mode is the default setting. To set the message mode to *mixed* on an interface:

{{< tabs "TabID494 ">}}
{{< tab "NVUE Commands ">}}

```
cumulus@switch:~$ nv set interface swp1 ptp message-mode mixed
cumulus@switch:~$ nv config apply
```

{{< /tab >}}
{{< tab "Linux Commands ">}}

Edit the `Default interface options` section of the  `/etc/ptp4l.conf` file to change the `message_mode` setting for the interface, then restart the `ptp4l` and `phc2sys` daemons.

```
cumulus@switch:~$ sudo nano /etc/ptp4l.conf
...
# Default interface options
#
time_stamping           software

# Interfaces in which ptp should be enabled
# these interfaces should be routed ports
# if an interface does not have an ip address
# the ptp4l will not work as expected.

[swp1]
logAnnounceInterval     0
logSyncInterval         -3
logMinDelayReqInterval  -3
announceReceiptTimeout  3
udp_ttl                 20
masterOnly              1
delay_mechanism         E2E
network_transport       UDPv4
message_mode            mixed
...
```

```
cumulus@switch:~$ sudo systemctl restart ptp4l.service phc2sys.service
```

{{< /tab >}}
{{< /tabs >}}

### TTL for a PTP Message

To restrict the number of hops a PTP message can travel, set the TTL on the PTP interface. You can set a value between 1 and 255.

{{< tabs "TabID462 ">}}
{{< tab "NVUE Commands ">}}

```
cumulus@switch:~$ nv set interface swp1 ptp ttl 20
cumulus@switch:~$ nv config apply
```

{{< /tab >}}
{{< tab "Linux Commands ">}}

Edit the `Default interface options` section of the  `/etc/ptp4l.conf` file to change the `udp_ttl` setting for the interface, then restart the `ptp4l` and `phc2sys` daemons.

```
cumulus@switch:~$ sudo nano /etc/ptp4l.conf
...
# Default interface options
#
time_stamping           software

# Interfaces in which ptp should be enabled
# these interfaces should be routed ports
# if an interface does not have an ip address
# the ptp4l will not work as expected.

[swp1]
logAnnounceInterval     0
logSyncInterval         -3
logMinDelayReqInterval  -3
announceReceiptTimeout  3
udp_ttl                 20
masterOnly              1
delay_mechanism         E2E
network_transport       UDPv4
...
```

```
cumulus@switch:~$ sudo systemctl restart ptp4l.service phc2sys.service
```

{{< /tab >}}
{{< /tabs >}}

### Acceptable Master Table

The acceptable master table option is a security feature that prevents a rogue player from pretending to be the Grandmaster to take over the PTP network. To use this feature, you configure the clock IDs of known Grandmasters in the acceptable master table and set the acceptable master table option on a PTP port. The BMC algorithm checks if the Grandmaster received on the Announce message is in this table before proceeding with the master selection. This option is disabled by default on PTP ports.

The following example command adds the Grandmaster clock ID 248a07.fffe.f41606 to the acceptable master table.

```
cumulus@switch:~$ nv set service ptp 1 acceptable-master 248a07.fffe.f41606
cumulus@switch:~$ nv config apply
```

You can also configure an alternate priority 1 value for the Grandmaster:

```
cumulus@switch:~$ nv set service ptp 1 acceptable-master 248a07.fffe.f41606 alt-priority 2
```

The following example commands enable the PTP acceptable master table option for swp1:

```
cumulus@switch:~$ nv set interface swp1 ptp acceptable-master on
cumulus@switch:~$ nv config apply
```

### PTP Timers

You can set the following timers for PTP messages.

| Timer | Description |
| ----- | ----------- |
| `announce-interval` | The average interval between successive Announce messages. Specify the value as a power of two in seconds. |
| `announce-timeout` | The number of announce intervals that have to occur without receiving an Announce message before a timeout occurs. <br>Make sure that this value is longer than the announce-interval in your network.|
| `delay-req-interval` | The minimum average time interval allowed between successive Delay Required messages. |
| `sync-interval` | The interval between PTP synchronization messages on an interface. Specify the value as a power of two in seconds. |

- To set the timers with NVUE, run the `nv set interface <interface> ptp timers <timer> <value>` command.
- To set the timers with Linux commands, edit the `/etc/ptp4l.conf` file and set the timers in the `Default interface options` section.

{{< tabs "TabID542 ">}}
{{< tab "NVUE Commands ">}}

The following example sets the announce interval between successive Announce messages on swp1 to -1.

```
cumulus@switch:~$ nv set interface swp1 ptp timers announce-interval -1
cumulus@switch:~$ nv config apply
```

The following example sets the mean sync-interval for multicast messages on swp1 to -5.

```
cumulus@switch:~$ nv set interface swp1 ptp timers sync-interval -5
cumulus@switch:~$ nv config apply
```

{{< /tab >}}
{{< tab "Linux Commands ">}}

Edit the `Default interface options` section of the `/etc/ptp4l.conf` file:

- To set the announce interval between successive Announce messages on swp1 to -1, change the `logAnnounceInterval` setting for the interface to -1.
- To set the mean sync-interval for multicast messages on swp1 to -5, change the `logSyncInterval` setting for the interface to -5.

After you edit the `/etc/ptp4l.conf` file, restart the `ptp4l` and `phc2sys` daemons.

```
cumulus@switch:~$ sudo nano /etc/ptp4l.conf
...
# Default interface options
#
time_stamping           software

# Interfaces in which ptp should be enabled
# these interfaces should be routed ports
# if an interface does not have an ip address
# the ptp4l will not work as expected.

[swp1]
logAnnounceInterval     -1
logSyncInterval         -5
logMinDelayReqInterval  -3
announceReceiptTimeout  3
udp_ttl                 20
masterOnly              1
delay_mechanism         E2E
network_transport       UDPv4
...
```

```
cumulus@switch:~$ sudo systemctl restart ptp4l.service phc2sys.service
```

{{< /tab >}}
{{< /tabs >}}

## Delete PTP Configuration

To delete PTP configuration, delete the PTP master and slave interfaces. The following example commands delete the PTP interfaces `swp1`, `swp2`, and `swp3`.

{{< tabs "TabID602 ">}}
{{< tab "NVUE Commands ">}}

```
cumulus@switch:~$ nv unset interface swp1 ptp
cumulus@switch:~$ nv unset interface swp2 ptp
cumulus@switch:~$ nv unset interface swp3 ptp
cumulus@switch:~$ nv config apply
```

{{< /tab >}}
{{< tab "Linux Commands ">}}

Edit the `/etc/ptp4l.conf` file to remove the interfaces from the `Default interface options` section, then restart the `ptp4l` and `phc2sys` daemons.

```
cumulus@switch:~$ sudo nano /etc/ptp4l.conf
...
# Default interface options
#
time_stamping           software

# Interfaces in which ptp should be enabled
# these interfaces should be routed ports
# if an interface does not have an ip address
# the ptp4l will not work as expected.
```

```
cumulus@switch:~$ sudo systemctl restart ptp4l.service phc2sys.service
```

{{< /tab >}}
{{< /tabs >}}

To disable PTP on the switch and stop the `ptp4l` and `phc2sys` processes:

{{< tabs "TabID627 ">}}
{{< tab "NVUE Commands ">}}

```
cumulus@switch:~$ nv set service ptp 1 enable off
cumulus@switch:~$ nv config apply
```

{{< /tab >}}
{{< tab "Linux Commands ">}}

```
cumulus@switch:~$ sudo systemctl stop ptp4l.service phc2sys.service
cumulus@switch:~$ sudo systemctl disable ptp4l.service phc2sys.service
```

{{< /tab >}}
{{< /tabs >}}

## Troubleshooting

NVUE provides several show commands for PTP. You can view the current PTP configuration, monitor violations, and see time attributes of the clock. For example, to show a summary of the PTP configuration on the switch, run the `nv show service ptp <instance>` command:

```
cumulus@switch:~$ nv show service ptp 1

                       operational  applied   description
----------------------  -----------  --------  ----------------------------------------------------------------------
enable                  off          on        Turn the feature 'on' or 'off'.  The default is 'off'.
clock-mode                           boundary  Clock mode
domain                               3         Domain number of the current syntonization
ipv4-dscp                            43        Sets the Diffserv code point for all PTP packets originated locally.
message-mode                         mixed     Mode in which PTP delay message is transmitted.
priority1                            254       Priority1 attribute of the local clock
priority2                            254       Priority2 attribute of the local clock
two-step                             off       Determines if the Clock is a 2 step clock
monitor
  max-offset-threshold               200       Maximum offset threshold in nano seconds
  min-offset-threshold               -200      Minimum offset threshold in nano seconds
  path-delay-threshold               1         Path delay threshold in nano seconds
...
```

To see the list of NVUE show commands for PTP, run `nv list-commands service ptp`:

```
cumulus@leaf01:mgmt:~$ nv list-commands service ptp
nv show service ptp
nv show service ptp <instance-id>
nv show service ptp <instance-id> acceptable-master
nv show service ptp <instance-id> acceptable-master <clock-id>
nv show service ptp <instance-id> monitor
nv show service ptp <instance-id> monitor violations
nv show service ptp <instance-id> monitor violations forced-master
nv show service ptp <instance-id> monitor violations forced-master <clock-id>
nv show service ptp <instance-id> monitor violations acceptable-master
nv show service ptp <instance-id> monitor violations acceptable-master <clock-id>
nv show service ptp <instance-id> current
nv show service ptp <instance-id> clock-quality
nv show service ptp <instance-id> parent
nv show service ptp <instance-id> parent grandmaster-clock-quality
...
```

To view PTP status information, including the delta in nanoseconds from the master clock:

```
cumulus@switch:~$ sudo pmc -u -b 0 'GET TIME_STATUS_NP'
sending: GET TIME_STATUS_NP
    7cfe90.fffe.f56dfc-0 seq 0 RESPONSE MANAGEMENT TIME_STATUS_NP
        master_offset              12610
        ingress_time               1525717806521177336
        cumulativeScaledRateOffset +0.000000000
        scaledLastGmPhaseChange    0
        gmTimeBaseIndicator        0
        lastGmPhaseChange          0x0000'0000000000000000.0000
        gmPresent                  true
        gmIdentity                 000200.fffe.000005
    000200.fffe.000005-1 seq 0 RESPONSE MANAGEMENT TIME_STATUS_NP
        master_offset              0
        ingress_time               0
        cumulativeScaledRateOffset +0.000000000
        scaledLastGmPhaseChange    0
        gmTimeBaseIndicator        0
        lastGmPhaseChange          0x0000'0000000000000000.0000
        gmPresent                  false
        gmIdentity                 000200.fffe.000005
    000200.fffe.000006-1 seq 0 RESPONSE MANAGEMENT TIME_STATUS_NP
        master_offset              5544033534
        ingress_time               1525717812106811842
        cumulativeScaledRateOffset +0.000000000
        scaledLastGmPhaseChange    0
        gmTimeBaseIndicator        0
        lastGmPhaseChange          0x0000'0000000000000000.0000
        gmPresent                  true
        gmIdentity                 000200.fffe.000005
```

## Example Configuration

In the following example, the boundary clock on the switch receives time from Master 1 (the grandmaster) on PTP slave port swp1, sets its clock and passes the time down through PTP master ports swp2, swp3, and swp4 to the hosts that receive the time.

{{< img src = "/images/cumulus-linux/date-time-ptp-config.png" >}}

This is the configuration for the above example. The example assumes that you have already configured the layer 3 routed interfaces (`swp1`, `swp2`, `swp3`, and `swp4`) you want to use for PTP.

{{< tabs "518 ">}}
{{< tab "NVUE Commands ">}}

```
cumulus@switch:~$ nv set service ptp 1 enable on
cumulus@switch:~$ nv set service ptp 1 priority2 254
cumulus@switch:~$ nv set service ptp 1 priority1 254
cumulus@switch:~$ nv set service ptp 1 domain 3
cumulus@switch:~$ nv set interface swp1 ptp enable on
cumulus@switch:~$ nv set interface swp2 ptp enable on
cumulus@switch:~$ nv set interface swp3 ptp enable on
cumulus@switch:~$ nv set interface swp4 ptp enable on
cumulus@switch:~$ nv config apply
```

{{< /tab >}}
{{< tab "/etc/nvue.d/startup.yaml file ">}}

```
- set:
    interface:
      lo:
        ip:
          address:
            10.10.10.1/32: {}
        type: loopback
      swp1:
        type: swp
        service:
          ptp:
            enable: on
      swp2:
        type: swp
        service:
          ptp:
            enable: on
      swp3:
        type: swp
        service:
          ptp:
            enable: on
      swp4:
        type: swp
        service:
          ptp:
            enable: on
    service:
      ptp:
        '1':
          enable: on
          priority1: 254
          priority2: 254
          domain: 3
```

{{< /tab >}}
{{< tab "/etc/ptp4l.conf file ">}}

```
cumulus@leaf02:mgmt:~$ sudo cat /etc/ptp4l.conf
...
[global]
#
# Default Data Set
#
slaveOnly               0
priority1               254
priority2               254
domainNumber            3

clock_type              BC

twoStepFlag             0
dscp_event              43
dscp_general            43

offset_from_master_min_threshold   -200
offset_from_master_max_threshold   200
mean_path_delay_threshold          1

#
# Run time options
#
logging_level           6
path_trace_enabled      0
use_syslog              1
verbose                 0
summary_interval        0

#
# Default interface options
#
time_stamping           software

# Interfaces in which ptp should be enabled
# these interfaces should be routed ports
# if an interface does not have an ip address
# the ptp4l will not work as expected.

[swp1]
logAnnounceInterval     0
logSyncInterval         -3
logMinDelayReqInterval  -3
announceReceiptTimeout  3
udp_ttl                 1
masterOnly              0
delay_mechanism         E2E
network_transport       UDPv4

[swp2]
logAnnounceInterval     0
logSyncInterval         -3
logMinDelayReqInterval  -3
announceReceiptTimeout  3
udp_ttl                 1
masterOnly              0
delay_mechanism         E2E
network_transport       UDPv4

[swp3]
logAnnounceInterval     0
logSyncInterval         -3
logMinDelayReqInterval  -3
announceReceiptTimeout  3
udp_ttl                 1
masterOnly              0
delay_mechanism         E2E
network_transport       UDPv4

[swp4]
logAnnounceInterval     0
logSyncInterval         -3
logMinDelayReqInterval  -3
announceReceiptTimeout  3
udp_ttl                 1
masterOnly              0
delay_mechanism         E2E
network_transport       UDPv4
```

{{< /tab >}}
{{< /tabs >}}
