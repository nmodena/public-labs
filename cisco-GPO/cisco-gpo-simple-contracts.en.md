# GPO Configuration - Advanced School of Networking

Example of GBP/GPO (Group Based Policy/Group Policy Options) Configuration for Simple Micro-segmentation in the Data Center

Date: 05/2026
Release: 1.0
Author: Nicola Modena - CCIE #19119 - JNCIE #986

- [GPO Configuration - Advanced School of Networking](#gpo-configuration---advanced-school-of-networking)
  - [Summary](#summary)
  - [Laboratory](#laboratory)
  - [Physical Topology](#physical-topology)
  - [Micro Segmentation Requirements](#micro-segmentation-requirements)
    - [Step 1 | Configure the system template](#step-1--configure-the-system-template)
    - [Step-2 | Defining Security Groups](#step-2--defining-security-groups)
      - [Check Type-2 MAX-IP ESG/PCTAG assignment.](#check-type-2-max-ip-esgpctag-assignment)
      - [Check Type-5 IP-PREFIX ESG/PCTAG assignment](#check-type-5-ip-prefix-esgpctag-assignment)
    - [Step 3 | Defining Services and Security Policies](#step-3--defining-services-and-security-policies)
    - [Step 4 | Defining Security Contacts](#step-4--defining-security-contacts)
  - [Security groups within VLANs](#security-groups-within-vlans)
    - [Identification of additional security groups](#identification-of-additional-security-groups)
    - [Creating security policies for additional security groups](#creating-security-policies-for-additional-security-groups)
  - [overall security policy](#overall-security-policy)

## Summary

The goal is to apply a micro-segmentation solution that separates the Front-End, Back-End, and Database components of an application pool hosted in the same VRF in the datacenter, but in separate VLANs.

Related Documentation: [Micro-segmentation for VXLAN fabrics using group policy option (GPO)](https://www.cisco.com/c/en/us/td/docs/dcn/nx-os/nexus9000/106x/configuration/vxlan/cisco-nexus-9000-series-nx-os-vxlan-configuration-guide-release-106x/microsegmentation_for_vxlan_fabrics_using_gpo.html)


## Laboratory

The laboratory consists of a NEXUS 9000v mini fabric (2 Leaf/2 Spine) and a pair of firewalls (active/standby) simulated with an IOSv Router.

![Physical Lab Topology](img/GPO-LAB-PHY.svg)

The NXOS version used for Nexus 9000v is 10.5.4

```sh
nx-01# sh ver
Cisco Nexus Operating System (NX-OS) Software
Nexus 9000v is a demo version of the Nexus Operating System
Software
  BIOS: version
  NXOS: version 10.5(4) [Maintenance Release]
  BIOS compile time:
  NXOS image file is: bootflash:///nxos64-cs.10.5.4.M.bin
  NXOS compile time:  10/10/2025 31:00:00 [10/16/2025 21:28:37]
```

Disclaimer 1: The virtual version of Nexus **DOES NOT WORK POLICY ENFORCEMENT IS NOT CORRECTLY SETTING GBP** in VXLAN packets. This can only be used to verify dataplane configuration and signaling.

Note: In the “containerized” version of the Nexus 9000v used in Containerlab, it is not possible to modify the system template. You must use a VM version in EVE-NG or GNS-Lab of CML.



## Physical Topology

The lab uses a DATA16 VRF composed of 4 VLANs, each hosting the Front End, Application Server, and Database.

![Tenant Configuration](img/GPO-LAB-VRF.svg)

This is the VLAN allocation and addressing plan:

vrf: DATA16 l3vni: 30016 (vlan3016)

| vlan |  vni  |    prefix    |  use        |
|:----:|:-----:|:------------:|:-----------:|
|  16  | 10016 | 10.0.16.0/24 | Front-End   |
|  17  | 10017 | 10.0.17.0/24 | Application |
|  18  | 10018 | 10.0.18.0/24 | Application |
|  19  | 10019 | 10.0.19.0/24 | Data-Base   |


## Micro Segmentation Requirements

As part of the process to increase internal security, the CISO requires the implementation of a micro-segmentation policy within the data center. This policy separates the Front-End, Application, and Database components, enabling only the protocols required for operation to communicate between them.
Specifically, the front ends perform a Layer-7 check and communicate with the application servers via HTTPS. The databases are MySQL and are accessed exclusively by the application servers.
Access to the VRF for publishing services and for infrastructure management is controlled by a border firewall.

### Step 1 | Configure the system template

To use GPOs, you must modify the switch’s TCAM allocation to dedicate resources to security features.
This is also reported when attempting to load the security groups feature:

```sh
nx-01(config)# feature security-group
Configured/Applied routing mode template is not security-groups
```

It is therefore necessary to specify the specific template and, in this case, also repartition the file system.
After completing this operation, the switch reboots.

```sh
nx-01(config)# system routing template-security-groups
Warning: This routing template is required for feature security-group. Please ensure the feature is enabled after applying routing mode.
Warning: This routing template requires extended SSD repartioning.
Please execute 'copy running-config startup-config' and 'system flash sda resize extended'
Warning: The command will take effect after next reload.

nx-01(config)# copy ru sta
[########################################] 100%
Copy complete, now saving to disk (please wait)...
Copy complete.

nx-01(config)# system flash sda resize extended

 lcm_cli_flash_resize SSD size is 64G. SSD resize is not supported on this device
nx-01(config)# reload
```

After rebooting, you can load the feature that enables GPO.

```sh
nx-01(config)# feature security-group
```

And you can check which leafs have GPO features enabled.

```sh
NX-01# sh nve peers detail
Details of nve Peers:
----------------------------------------
Peer-Ip: 10.100.0.2
    NVE Interface       : nve1
    Peer State          : Up
    Peer Uptime         : 00:33:44
    Router-Mac          : 5004.0000.1b08
    Peer First VNI      : 30016
    Time since Create   : 00:33:44
    Configured VNIs     : 10016-10019,30016
    Provision State     : peer-add-complete
    Learnt CP VNIs      : 10016-10019,30016
    vni assignment mode : SYMMETRIC
    Peer Location       : N/A
    Group policy capable: yes                       <--------------------
----------------------------------------
```

NOTE: Enabling the GPO is done by recognizing that the PCTAG community is now added to type-2, type-3, and type-5 announcements (usually with tag 0 = no group).

Related Documentation: [Enable GPO](https://www.cisco.com/c/en/us/td/docs/dcn/nx-os/nexus9000/106x/configuration/vxlan/cisco-nexus-9000-series-nx-os-vxlan-configuration-guide-release-106x/microsegmentation_for_vxlan_fabrics_using_gpo.html#task_m2r_ckn_cbc)

### Step-2 | Defining Security Groups

To comply with internal segregation guidelines, three security groups are provided for internal segregation, while an external security group is assigned to traffic controlled by the firewall.

![Security Group Assignement](img/GPO-LAB-ESG.svg)

Thanks to a well-organized grouping of components into different VLANs, security groups are mapped to VLANS and merge some of them. In fact, the security group for applications involves two VLANs. This allows for the abstraction of security groups, making them independent of their content and the selection mechanism used to specify them.

```sh
security-group 100 name FE
  type layer4-7
  match vlan 16
!
security-group 200 name APP
  type layer4-7
  match vlan 17,18
!
security-group 300 name DB
  type layer4-7
  match vlan 19
!
```

Security Groups can also be assigned to security zones outside the EVPN/VXLAN domain. In this case, the default route representing all traffic destined for the firewall is simply chosen. Enforcement policies to/from the VRF will then be delegated and enforced by the firewall in the traditional manner.

```sh
security-group 400 name OUTSIDE
  type layer4-7
  match external-subnets vrf DATA16 ipv4 0.0.0.0/0
!
```

This way, the policy will be applied to the routing function, which will use the default route as the destination. Routing operations within the VRF will also use the specific prefix mappings associated with the VLANs. This means, for example, that the /24 prefixes associated with the various internal VLANs automatically assume the respective security groups.

#### Check Type-2 MAX-IP ESG/PCTAG assignment.

host 10.0.16.161 in Front-End vlan 16 is identified in Security Group 100 (community PCTAG:0:0:100)

```sh
NX-01# sh bgp l2vpn evpn 10.0.16.161
BGP routing table information for VRF default, address family L2VPN EVPN
Route Distinguisher: 10.100.0.1:32783    (L2VNI 10016)
BGP routing table entry for [2]:[0]:[0]:[48]:[5001.0007.0000]:[32]:[10.0.16.161]/272, version 298
Paths: (1 available, best #1)
Flags: (0x000102) (high32 00000000) on xmit-list, is not in l2rib/evpn

  Advertised path-id 1
  Path type: local, path is valid, is best path, no labeled nexthop
  AS-Path: NONE, path locally originated
    10.100.0.1 (metric 0) from 0.0.0.0 (10.100.0.1)
      Origin IGP, MED not set, localpref 100, weight 32768
      Received label 10016 30016
      Extcommunity: RT:65000:10016 RT:65000:30016 ENCAP:8 PCTAG:0:0:100         <-------------
          Router MAC:5003.0000.1b08
```

#### Check Type-5 IP-PREFIX ESG/PCTAG assignment

Outbound traffic that will use the default route is identified by Security Group 400 (community PCTAG:0:0:400)

```sh
NX-01# sh bgp l2vpn evpn 0.0.0.0 vrf DATA16
BGP routing table information for VRF default, address family L2VPN EVPN
Route Distinguisher: 10.100.0.1:4    (L3VNI 30016)
BGP routing table entry for [5]:[0]:[0]:[0]:[0.0.0.0]/224, version 256
Paths: (1 available, best #1)
Flags: (0x000002) (high32 00000000) on xmit-list, is not in l2rib/evpn

  Advertised path-id 1
  Path type: local, path is valid, is best path, no labeled nexthop
  Gateway IP: 10.0.99.2
  AS-Path: 65400 , path sourced external to AS
    10.100.0.1 (metric 0) from 0.0.0.0 (10.100.0.1)
      Origin IGP, MED not set, localpref 110, weight 0
      Received label 30016
      Received path-id 1
      Extcommunity: RT:65000:30016 ENCAP:8 PCTAG:0:0:400 Router MAC:5003.0000.1b08  <------------
```

### Step 3 | Defining Services and Security Policies

Three types of traffic are allowed among the security groups: HTTPS and MySQL. Given that control has been delegated to the firewall, all IP traffic is sent.
This is equivalent to defining a service (TCP/UDP/etc) on a firewall.

```sh
class-map type security match-any HTTPS
  match ipv4 tcp stateful dport 443
!
class-map type security match-any MYSQL
  match ipv4 tcp stateful dport 3306
!
class-map type security match-any ANY
  match ipv4  
```


The *statefull* specification available for TCP services is actually a stateless check, but it checks the SYN/ACK flags, allowing sessions to be established only in the indicated direction. It is not possible to open TCP sessions in the opposite direction by specifying 443 or 3306 as the source port.

Security policies define which services you want enabled between the different security groups. Possible actions are accept (default), deny, and redirect, and you can also optionally specify the log.

```sh
policy-map type security PERMIT-HTTPS
  class HTTPS
!
policy-map type security PERMIT-MYSQL
  class MYSQL
!
policy-map type security PERMIT-ANY
  class ANY
```


### Step 4 | Defining Security Contacts

Policy enforcement: Security policies that control traffic between the different security groups are defined at the VRF level. In a Cisco environment, they are called contracts.

The following is established:

- Only HTTPS traffic is allowed from the Front End to the Applications.
- Only applications can communicate with the database security group using the specific protocol.
- The any selector can be used to define a generic policy to/from SG 400.

```sh
vrf context DATA16
  security contract source 100 destination 200 policy PERMIT-HTTPS bidir
  security contract source 200 destination 300 policy PERMIT-MYSQL bidir
  security contract source any destination 400 policy PERMIT-ANY bidir
```

Policies are automatically generated bidirectionally. For example, in the case of HTTPS, they are developed as:

```text
SG 100 TCP HIGH PORT ( SYN without ACK )  to SG 200 TCP PORT 443
SG 200 TCP port 443 ( ACK )               to SG 100 TCP HIGH PORT 
```

For IP traffic, the application direction is irrelevant; the rule for group 400 could also be written by reversing the members. In this case, security policies, including stateful inspection, are delegated to the firewall.

It is now necessary to enable policy enforcement by **selecting the enforcement mode for traffic between security groups where a policy/contract has not been explicitly defined, and the fallback for policy that does not define all the traffic rules;** this can be either *deny* or *accept*.

```sh
vrf context DATA16
  security enforce tag 999 default deny
```

At the same time, a dedicated Security Group ID must be specified for managing Broadcast/Multicast traffic (and traffic not matched by a classification rule), which is not enforced but remains specific to the VRF.

## Security groups within VLANs

Due to the activation of a new payment system, some application servers and databases will need to be segregated within a new perimeter subject to PCI regulations. The request is for the operation to be as minimally invasive as possible; we do not want to change the VLAN allocation, the VRF addressing plan, or even renumber the Application Servers and Databases.

### Identification of additional security groups

To minimize impact, two additional security groups and IP-based mapping will be implemented to segregate the new PCI perimeter.

The two Application Servers have addresses 10.0.17.171 and 10.0.18.182.
The two Database Servers have addresses 10.0.19.193 and 10.0.19.194.

Two new security groups are therefore defined using a mapping based on member IP addresses.

![Security Group PCI](img/GPO-LAB-PCI.svg)

```sh
security-group 201 name APP-PCI
  type layer4-7
  match connected-endpoints vrf DATA16 ipv4 10.0.17.171/32
  match connected-endpoints vrf DATA16 ipv4 10.0.18.182/32
!
security-group 301 name DB-PCI
  type layer4-7
  match connected-endpoints vrf DATA16 ipv4 10.0.19.193/32
  match connected-endpoints vrf DATA16 ipv4 10.0.19.194/32
!
```

The IP-based mapping takes precedence over the VLAN-based mapping, and the identified hosts are associated with the new security groups.

```sh
NX-01# sh bgp l2vpn evpn 10.0.17.171
BGP routing table information for VRF default, address family L2VPN EVPN
Route Distinguisher: 10.100.0.1:32784    (L2VNI 10017)
BGP routing table entry for [2]:[0]:[0]:[48]:[5001.0008.0000]:[32]:[10.0.17.171]/272, version 300
Paths: (1 available, best #1)
Flags: (0x000102) (high32 00000000) on xmit-list, is not in l2rib/evpn

  Advertised path-id 1
  Path type: local, path is valid, is best path, no labeled nexthop
  AS-Path: NONE, path locally originated
    10.100.0.1 (metric 0) from 0.0.0.0 (10.100.0.1)
      Origin IGP, MED not set, localpref 100, weight 32768
      Received label 10017 30016
      Extcommunity: RT:65000:10017 RT:65000:30016 ENCAP:8 PCTAG:0:0:201.   <----------------
          Router MAC:5003.0000.1b08
```

### Creating security policies for additional security groups

Now simply create new policies to allow HTTPS traffic from the front end to the PCI application servers, which, in turn, can access the respective PCI databases via the MySQL protocol.

```sh
vrf context DATA16
  security contract source 100 destination 201 policy PERMIT-HTTPS bidir
  security contract source 201 destination 301 policy PERMIT-MYSQL bidir
```

Using the *any* selector for firewall access and the “default deny” enforcement mode, respectively, ensure the firewall can be used to administer hosts and to inhibit lateral communication between all other security groups within the VRF.

## overall security policy

For completeness, the overall configuration of security groups and configured policies is shown.

```sh
security-group 100 name FE
  type layer4-7
  match vlan 16
!
security-group 200 name APP
  type layer4-7
  match vlan 17,18
!
security-group 201 name APP-PCI
  type layer4-7
  match connected-endpoints vrf DATA16 ipv4 10.0.17.171/32
  match connected-endpoints vrf DATA16 ipv4 10.0.18.182/32
!
security-group 300 name DB
  type layer4-7
  match vlan 19
!
security-group 301 name DB-PCI
  type layer4-7
  match connected-endpoints vrf DATA16 ipv4 10.0.19.193/32
  match connected-endpoints vrf DATA16 ipv4 10.0.19.194/32
!
security-group 400 name OUTSIDE
  type layer4-7
  match external-subnets vrf DATA16 ipv4 0.0.0.0/0
!
vrf context DATA16
  security contract source any destination 400 policy PERMIT-ANY
  security contract source 100 destination 200 policy PERMIT-HTTPS
  security contract source 100 destination 201 policy PERMIT-HTTPS
  security contract source 200 destination 300 policy PERMIT-MYSQL
  security contract source 201 destination 301 policy PERMIT-MYSQL
  security enforce tag 999 default deny
```


