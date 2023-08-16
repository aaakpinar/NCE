---
comments: true
tags:
  - evpn
  - multi-homing
  - ethernet-segments
---

<script type="text/javascript" src="https://cdn.jsdelivr.net/gh/hellt/drawio-js@main/embed2.js" async></script>

|                                |                                                                                                                                                                                                                                                                                                                                                                                                                                                                                             |
| ------------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Tutorial name**              | EVPN L2 Multi-homing with SR Linux                                                                                                                                                                                                                                                                                                                                                                                                                                                                 |
| **Lab components**             | 4 SR Linux nodes                                                                                                                                                                                                                                                                                                                                                                                                                                                                            |
| **Resource requirements**      | :fontawesome-solid-microchip: 2vCPU <br/>:fontawesome-solid-memory: 6 GB                                                                                                                                                                                                                                                                                                                                                                                                                    |
| **Containerlab topology file** | [evpn-mh01.clab.yml][topofile]                                                                                                                                                                                                                                                                                                                                                                                                                                                                 |
| **Lab name**                   | evpn-mh01                                                                                                                                                                                                                                                                                                                                                                                                                                                                                      |
| **Packet captures**            | [EVPN IMET routes exchange][capture-imets], [RT2 routes exchange with ICMP in datapath][capture-rt2-datapath]                                                                                                                                                                                                                                                                                                                                                                               |
| **Main ref documents**         | [RFC 7432 - BGP MPLS-Based Ethernet VPN](https://datatracker.ietf.org/doc/html/rfc7432)<br/>[RFC 8365 - A Network Virtualization Overlay Solution Using Ethernet VPN (EVPN)](https://datatracker.ietf.org/doc/html/rfc8365)<br/>[Nokia 7220 SR Linux Advanced Solutions Guide](https://documentation.nokia.com/srlinux/22-11/SR_Linux_Book_Files/Advanced_Solutions_Guide/evpn-l2-multihome.html)<br/>[Nokia 7220 SR Linux EVPN-VXLAN Guide](https://documentation.nokia.com/srlinux/22-11/title/evpn_vxlan.html) |
| **Version information**[^1]    | [`containerlab:0.42.0`][clab-install], [`srlinux:23.3.2`][srlinux-container], [`docker-ce:23.0.3`][docker-install]                                                                                                                                                                                                                                                                                                                                                                        |

One of the many advantages of EVPN is its built-in multi-homing (MH) capability, which is standards-based and defined by RFCs 7432, 8365. 

This tutorial will show the configuration items of L2 multi-homing provided by multiple PEs in an EVPN-based SR Linux fabric.

EVPN provides multi-homing with the Ethernet Segments (ES), which may be a new concept to some of the readers. Therefore, the terminology will also be discussed in the following chapters.

The lab comprises a spine, 3 leaf(PEs) routers, and two Alpine Linux hosts(CEs). A multi-homed CE is connected to leaf1, while another CE is connected to leaf3 for testing purposes.

<div class="mxgraph" style="max-width:100%;border:1px solid transparent;margin:0 auto; display:block;" data-mxgraph="{&quot;page&quot;:0,&quot;zoom&quot;:2,&quot;highlight&quot;:&quot;#0000ff&quot;,&quot;nav&quot;:true,&quot;check-visible-state&quot;:true,&quot;resize&quot;:true,&quot;url&quot;:&quot;https://github.com/aaakpinar/NCE/blob/evpn-mh/evpn-mh/evpn-mh-fabric.svg&quot;}"></div>

![image](https://github.com/aaakpinar/NCE/blob/evpn-mh/evpn-mh/evpn-mh-fabric.svg)

## Lab deployment








## EVPN Multi-homing Terminology

Before diving into the hands-on, I'll mention the terms that would help better understand the configurations.

+ **MAC-VRF:** A broadcast domain in SR Linux.

+ **Ethernet Segment (ES):** Defines the CE links connected to multiple PEs. An ES is configured in all PEs that a CE is connected and has a unique identifier (ESI) advertised via EVPN.

![image](https://github.com/aaakpinar/NCE/blob/evpn-mh/evpn-mh/esi.svg)

+ **Multi-homing Modes:** The standard defines two modes: single-active and all-active. Single-active mode has only one active link, while all-active mode uses all links and provides load balancing. This tutorial covers an example of all-active multi-homing.
![image](https://github.com/aaakpinar/NCE/blob/evpn-mh/evpn-mh/evpn-mh-all-active.svg)

+ **Link Aggregation Group (LAG):** A LAG is required for all-active but optional for single-active multi-homing.

The following procedures are essential to EVPN multi-homing but not a typical configuration item;

+ Designated Forwarder (DF): The leaf that is elected to forward BUM traffic. The election is based on the route-type 4 (RT4) exchange, known as the ES routes of EVPN.
+ Split-horizon (Local bias): A mechanism to avoid looping the BUM traffic received from the CE back to itself by a peer PE. Local bias is used for all-active and based on RT4 exchange. 
+ Aliasing: For remote PEs that are not part of ES to load-balance traffic to the multi-homed CE. RT1 (Auto-discovery) is advertised for aliasing.

EVPN route types 1 and 4 are used to implement the multi-homing procedures.

## SR Linux Multi-Homing Configurations

The following items must be configured in all PEs that provide multi-homing to a CE.

+ A LAG and member interfaces
+ Ethernet segment
+ MAC-VRF to interface association

The lab is pre-configured with underlay, EVPN routing with BGP, and a MAC-VRF for CE-to-CE L2 communication.
  >Check out _[L2 EVPN tutorial](https://learn.srlinux.dev/tutorials/l2evpn/evpn/#mac-vrf) to learn more about the pre-configured part!_ 

### LAG Configuration

LAG is required for the all-active mode but can be skipped for single-active cases.

In this example, a LAG is created for an all-active multi-homing mode. The target configuration between a multihomed CE and PEs is shown below.

**IMAGE LAG HERE**

The configuration snippet below shows a LAG with a subinterface and its LACP settings.

```
enter candidate
    /interface lag1 {
        admin-state enable
        vlan-tagging true
        subinterface 1 {
            type bridged
            vlan {
                encap {
                    untagged {
                    }
                }
            }
        }
        lag {
            lag-type lacp
            member-speed 10G
            lacp {
                interval SLOW
                lacp-mode ACTIVE
                admin-key 11
                system-id-mac 00:00:00:00:00:11
                system-priority 11
            }
        }
    }
```

The lag1 is created with the `vlan-tagging` enabled so that this LAG can have multiple subinterfaces with different VLAN tags. In this way, each subinterface can be attached to a different MAC-VRF. The subinterface 1 is created here with `untagged` (tag0) encapsulation.

The `lag type` can be LACP or static. Here it is configured as LACP, so its parameters must match in all nodes, leaf1 and leaf2 in this example.

And associate the physical interface(s) with the LAG to complete this part. 

```
enter candidate
    /interface ethernet-1/11 {
        admin-state enable
        ethernet {
            aggregate-id lag1
        }
    }
```

All PEs that offer a multi-homing to a CE must be configured similarly with the lag and interface configurations.

### Ethernet Segment Configuration

In SR Linux, the `ethernet-segments` are configured under [ system network-instance protocols evpn ] context.

```
enter candidate
/system network-instance protocols 
    evpn {
        ethernet-segments {
            bgp-instance 1 {
                ethernet-segment ES-1 {
                    admin-state enable
                    esi 01:11:11:11:11:11:11:00:00:01
                    multi-homing-mode all-active
                    interface lag1 {
                    }
                }
            }
        }
    }
    bgp-vpn {
        bgp-instance 1 {
        }
    }
```

An `ethernet-segment` is created with a name (ES-1) under the BGP-instance 1. The ES identifier (`esi`) and the `multi-homing-mode` must match in all leaf routers. At last, we associate the `lag1` interface with the ES-1.

Besides the ethernet segments, the `bgp-vpn` with `bgp-instance 1` is configured to get the ES routes' BGP information (RT/RD).

### MAC-VRF Interface Configuration

Typically, an L2 multi-homed LAG needs to be associated with a MAC-VRF.

```
enter candidate
    /network-instance mac-vrf-1 {
        interface lag1.1 {
        }
    }
```

The whole MAC-VRF with VXLAN configuration is covered [here](https://learn.srlinux.dev/tutorials/l2evpn/evpn/#mac-vrf).

With this, an all-active EVPN-MH configuration is completed.

Let's see the configuration example on the CE side.

## CE (Alpine Linux) Configuration

The ce2 is configured with a bond0 with slave interfaces eth1 and eth2. Similar to SR Linux part, it's configured with LACP (802.3ad).

The configuration file is /etc/network/interfaces and an example configuration snippet is below:

```
auto bond0
auto eth1
auto eth2

iface eth1 inet manual
bond-master bond0
mtu 1400

iface eth2 inet manual
bond-master bond0
mtu 1400

iface bond0 inet static
address 100.101.1.11
netmask 255.255.255.0
bond-mode 802.3ad
bond-xmit-hash-policy layer3+4
bond-miimon 300
mtu 1400
```

Similarly, ce2 is configured with an IP address but without bonding:

```
auto eth1
iface eth1 inet static
address 100.101.1.14
netmask 255.255.255.0
mtu 1400
```

## Verifying the Configurations






























<div class="mxgraph" style="max-width:100%;border:1px solid transparent;margin:0 auto; display:block;" data-mxgraph="{&quot;page&quot;:1,&quot;zoom&quot;:2,&quot;highlight&quot;:&quot;#0000ff&quot;,&quot;nav&quot;:true,&quot;check-visible-state&quot;:true,&quot;resize&quot;:true,&quot;url&quot;:&quot;https://raw.githubusercontent.com/srl-labs/learn-srlinux/diagrams/quickstart.drawio&quot;}"></div>

The tutorial will consist of the following major parts:

* [Fabric configuration](fabric.md) - here we will configure the routing protocol in the underlay of a fabric to advertise the Virtual Tunnel Endpoints (VTEP) of the leaf switches.
* [EVPN configuration](evpn.md) - this chapter is dedicated to the EVPN service configuration and validation.

## Lab deployment

To let you follow along the configuration steps of this tutorial we created a lab that you can deploy on any Linux VM:

The [containerlab file][topofile] that describes the lab topology is referenced below in full:

```yaml
--8<-- "labs/evpn01.clab.yml"
```

Save[^2] the contents of this file under `evpn01.clab.yml` name and you are ready to deploy:

```
$ containerlab deploy -t evpn01.clab.yml
INFO[0000] Containerlab v0.41.0 started                 
INFO[0000] Parsing & checking topology file: evpn01.clab.yml 
INFO[0005] Creating lab directory: /root/srl-labs/learn-srlinux/clab-evpn01 
INFO[0005] Creating container: "srv2"                   
INFO[0005] Creating container: "srv1"                   
INFO[0005] Creating container: "spine1"                 
INFO[0005] Creating container: "leaf1"                  
INFO[0005] Creating container: "leaf2"                  
INFO[0006] Creating virtual wire: srv2:eth1 <--> leaf2:e1-1 
INFO[0006] Creating virtual wire: leaf2:e1-49 <--> spine1:e1-2 
INFO[0006] Creating virtual wire: leaf1:e1-49 <--> spine1:e1-1 
INFO[0006] Creating virtual wire: srv1:eth1 <--> leaf1:e1-1 
INFO[0007] Running postdeploy actions for Nokia SR Linux 'spine1' node 
INFO[0007] Running postdeploy actions for Nokia SR Linux 'leaf1' node 
INFO[0007] Running postdeploy actions for Nokia SR Linux 'leaf2' node 
INFO[0021] Adding containerlab host entries to /etc/hosts file 
+---+--------------------+--------------+---------------------------------+-------+---------+-----------------+----------------------+
| # |        Name        | Container ID |              Image              | Kind  |  State  |  IPv4 Address   |     IPv6 Address     |
+---+--------------------+--------------+---------------------------------+-------+---------+-----------------+----------------------+
| 1 | clab-evpn01-leaf1  | c88892905319 | ghcr.io/nokia/srlinux:23.3.1    | srl   | running | 172.20.20.11/24 | 2001:172:20:20::b/64 |
| 2 | clab-evpn01-leaf2  | 55ab0f731815 | ghcr.io/nokia/srlinux:23.3.1    | srl   | running | 172.20.20.7/24  | 2001:172:20:20::7/64 |
| 3 | clab-evpn01-spine1 | 33565578fc32 | ghcr.io/nokia/srlinux:23.3.1    | srl   | running | 172.20.20.8/24  | 2001:172:20:20::8/64 |
| 4 | clab-evpn01-srv1   | 86a54b5d1999 | ghcr.io/hellt/network-multitool | linux | running | 172.20.20.10/24 | 2001:172:20:20::a/64 |
| 5 | clab-evpn01-srv2   | dee331143cda | ghcr.io/hellt/network-multitool | linux | running | 172.20.20.9/24  | 2001:172:20:20::9/64 |
+---+--------------------+--------------+---------------------------------+-------+---------+-----------------+----------------------+
```

A few seconds later containerlab finishes the deployment with providing a summary table that outlines connection details of the deployed nodes. In the "Name" column we have the names of the deployed containers and those names can be used to reach the nodes, for example to connect to the SSH of `leaf1`:

```bash
# default credentials admin:NokiaSrl1!
ssh admin@clab-evpn01-leaf1
```

With the lab deployed we are ready to embark on our learn-by-doing EVPN configuration journey!

!!!note
    We advise the newcomers not to skip the [SR Linux basic concepts](../../kb/hwtypes.md) chapter as it provides just enough[^4] details to survive in the configuration waters we are about to get.

[topofile]: https://github.com/srl-labs/learn-srlinux/blob/master/labs/evpn01.clab.yml
[clab-install]: https://containerlab.srlinux.dev/install/
[srlinux-container]: https://github.com/orgs/nokia/packages/container/package/srlinux
[docker-install]: https://docs.docker.com/engine/install/
[capture-imets]: https://github.com/srl-labs/learn-srlinux/blob/master/docs/tutorials/l2evpn/evpn01-imet-routes.pcapng
[capture-rt2-datapath]: https://github.com/srl-labs/learn-srlinux/blob/master/docs/tutorials/l2evpn/evpn01-macip-routes.pcapng

[^1]: the following versions have been used to create this tutorial. The newer versions might work, but if they don't, please pin the version to the mentioned ones.
[^2]: Or download it with `curl -LO https://github.com/srl-labs/learn-srlinux/blob/master/labs/evpn01.clab.yml`
[^3]: Per [RFC 8365](https://datatracker.ietf.org/doc/html/rfc8365) & [RFC 7432](https://datatracker.ietf.org/doc/html/rfc7432)
[^4]: For a complete documentation coverage don't hesitate to visit our [documentation portal](https://bit.ly/iondoc).
