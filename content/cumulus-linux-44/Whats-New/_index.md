---
title: What's New
author: NVIDIA
weight: 5
toc: 2
---
This document supports the Cumulus Linux 4.4 release, and lists new platforms and features.

- For a list of all the platforms supported in Cumulus Linux 4.4, see the {{<exlink url="www.nvidia.com/en-us/networking/ethernet-switching/hardware-compatibility-list/" text="Hardware Compatibility List (HCL)">}}.
- For a list of open and fixed issues in Cumulus Linux 4.4, see the {{<link title="Cumulus Linux 4.4 Release Notes" text="Cumulus Linux 4.4 Release Notes">}}.
- To upgrade to Cumulus Linux 4.4, follow the steps in {{<link url="Upgrading-Cumulus-Linux">}}.

## What's New in Cumulus Linux 4.4

Cumulus Linux 4.4 supports supports new platforms, provides bug fixes, and contains several new features and improvements.

### New Platforms

- Mellanox SN3700C-S (100G Spectrum-2) with Secure Boot

### New Features and Enhancements

- {{<link url="NVIDIA-User-Experience-NVUE" text="NVIDIA User Experience (NVUE)">}} is a new object-oriented, schema driven model of a complete Cumulus Linux system (hardware and software) with a robust API that allows multiple interfaces to both view and configure any element within the system. NVUE is an early access feature currently in BETA and open to customer feedback.
- {{<link url="VLAN-aware-Bridge-Mode/" text="Multiple VLAN-aware bridges">}}
- {{<link url="VXLAN-Devices/#single-vxlan-device" text="Single VXLAN Devices">}}
- {{<link url="EVPN-Multihoming" text="EVPN multihoming Head End Replication">}}
- {{<link url="Precision-Time-Protocol-PTP" text="PTP Boundary Clock">}} enhancements
- {{<link url="Protocol-Independent-Multicast-PIM/#allow-rp" text="PIM Allow RP">}}
- {{<link url="Optional-BGP-Configuration/#conditional-advertisement" text="BGP conditional route advertisement">}}
- {{<link url="IGMP-and-MLD-Snooping/#optimized-multicast-flooding-omf" text="Optimized Multicast Flooding (OMF)">}}
- Smart System Manager {{<link url="Smart-System-Manager" text="warm boot">}}
- {{<link url="Quality-of-Service" text="QoS enhancements ">}} (`traffic.conf` and `datapath.conf` files removed and replaced)
- {{<link url="Hybrid-Cloud-Connectivity-with-QinQ-and-VXLANs" text="QinQ Double-tagged translation ">}} is now supported on NVIDIA Spectrum-2 and above
- A specific software license key is no longer required to enable the `switchd` service

### Unsupported Platforms

Cumulus Linux 4.4 supports NVIDIA Spectrum-based ASIC platforms only. This release removes support for Broadcom-based networking ASICs. Broadcom-based ASICs will continue to be supported throughout the life of the Cumulus Linux 3.7 and 4.3 releases.
