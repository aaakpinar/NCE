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

This tutorial will help you learn about L2 multi-homing with EVPN and guide you on how to configure it in an EVPN-based SR Linux fabric.

EVPN provides multi-homing with the Ethernet Segments (ES), which may be a new concept to some readers. Therefore, the terminology will also be discussed in the following chapters.

The lab comprises a spine, three leaf(PEs) routers, and two Alpine Linux hosts(CEs). A multi-homed CE is connected to leaf1, while another is linked to leaf3 for testing purposes.

<div class="mxgraph" style="max-width:100%;border:1px solid transparent;margin:0 auto; display:block;" data-mxgraph="{&quot;page&quot;:0,&quot;zoom&quot;:2,&quot;highlight&quot;:&quot;#0000ff&quot;,&quot;nav&quot;:true,&quot;check-visible-state&quot;:true,&quot;resize&quot;:true,&quot;url&quot;:&quot;https://raw.githubusercontent.com/srl-labs/learn-srlinux/diagrams/evpn-mh.drawio&quot;}"></div>

## Lab deployment

As usual, this lab is powered by containerlab and can be deployed on any Linux VM with enough resources mentioned in the table at the beginning.

This lab comes with pre-configurations that are explained in [L2 EVPN tutorial](https://learn.srlinux.dev/tutorials/l2evpn/evpn/#mac-vrf), which is highly recommended if you haven't played with SR Linux or EVPN yet.

The topology and pre-configurations are defined in the containerlab topology file.

The SR Linux configurations are referred to as [config files][configs] (.cfg), and Alpine Linux configurations are defined in the [topology file][topofile] to be directly executed at the deployment.

```yaml
--8<-- "labs/evpn-mh/evpn-mh01.clab.yml"
```

The SR Linux configurations are 
=== "leaf1"
    ```yaml
    --8<-- "labs/evpn-mh/leaf1.cfg"
    ```
=== "leaf2"
    ```yaml
    --8<-- "labs/evpn-mh/leaf2.cfg"
    ```
=== "leaf3"
    ```yaml
    --8<-- "labs/evpn-mh/leaf3.cfg"
    ```
=== "spine1"
    ```yaml
    --8<-- "labs/evpn-mh/spine1.cfg"
    ```

Save [these][path-evpn-mh] to your Linux machine and deploy:

```
# containerlab deploy -t evpn-mh01.clab.yml
[root@clab-vm1 evpn-mh01]# containerlab deploy
INFO[0000] Containerlab v0.44.0 started
INFO[0000] Parsing & checking topology file: evpn-mh01.clab.yml
INFO[0000] Creating docker network: Name="clab", IPv4Subnet="172.20.20.0/24", IPv6Subnet="2001:172:20:20::/64", MTU="1500"
WARN[0000] Unable to load kernel module "ip_tables" automatically "load ip_tables failed: exec format error"
INFO[0000] Creating lab directory: /root/demo/learn.srlinux/clab/evpn-mh01/clab-evpn-mh01
WARN[0000] SSH_AUTH_SOCK not set, skipping pubkey fetching
INFO[0000] Creating container: "ce2"
INFO[0000] Creating container: "ce1"
INFO[0000] Creating container: "spine1"
INFO[0000] Creating container: "leaf3"
INFO[0000] Creating container: "leaf1"
INFO[0000] Creating container: "leaf2"
INFO[0003] Creating link: leaf1:e1-49 <--> spine1:e1-1
INFO[0003] Creating link: ce1:eth1 <--> leaf1:e1-1
INFO[0003] Creating link: leaf3:e1-49 <--> spine1:e1-3
INFO[0003] Creating link: leaf2:e1-49 <--> spine1:e1-2
INFO[0003] Creating link: ce2:eth1 <--> leaf3:e1-1
INFO[0003] Creating link: ce1:eth2 <--> leaf2:e1-1
INFO[0004] Creating link: ce2:eth2 <--> leaf3:e1-2
INFO[0004] Creating link: ce2:eth3 <--> leaf3:e1-3
INFO[0004] Running postdeploy actions for Nokia SR Linux 'leaf3' node
INFO[0004] Running postdeploy actions for Nokia SR Linux 'leaf1' node
INFO[0004] Running postdeploy actions for Nokia SR Linux 'spine1' node
INFO[0004] Running postdeploy actions for Nokia SR Linux 'leaf2' node
INFO[0025] Adding containerlab host entries to /etc/hosts file
INFO[0026] Executed command "ip link add bond0 type bond mode 802.3ad" on the node "ce1". stdout:
INFO[0026] Executed command "ip link set address 00:c1:ab:00:00:11 dev bond0" on the node "ce1". stdout:
INFO[0026] Executed command "ip addr add 192.168.0.11/24 dev bond0" on the node "ce1". stdout:
INFO[0026] Executed command "ip link set eth1 down" on the node "ce1". stdout:
INFO[0026] Executed command "ip link set eth2 down" on the node "ce1". stdout:
INFO[0026] Executed command "ip link set eth1 master bond0" on the node "ce1". stdout:
INFO[0026] Executed command "ip link set eth2 master bond0" on the node "ce1". stdout:
INFO[0026] Executed command "ip link set eth1 up" on the node "ce1". stdout:
INFO[0026] Executed command "ip link set eth2 up" on the node "ce1". stdout:
INFO[0026] Executed command "ip link set bond0 up" on the node "ce1". stdout:
INFO[0026] Executed command "ip link set address 00:c1:ab:00:00:21 dev eth1" on the node "ce2". stdout:
INFO[0026] Executed command "ip link set address 00:c1:ab:00:00:22 dev eth2" on the node "ce2". stdout:
INFO[0026] Executed command "ip link set address 00:c1:ab:00:00:23 dev eth3" on the node "ce2". stdout:
INFO[0026] Executed command "ip link add dev vrf-1 type vrf table 1" on the node "ce2". stdout:
INFO[0026] Executed command "ip link set dev vrf-1 up" on the node "ce2". stdout:
INFO[0026] Executed command "ip link set dev eth1 master vrf-1" on the node "ce2". stdout:
INFO[0026] Executed command "ip link add dev vrf-2 type vrf table 2" on the node "ce2". stdout:
INFO[0026] Executed command "ip link set dev vrf-2 up" on the node "ce2". stdout:
INFO[0026] Executed command "ip link set dev eth2 master vrf-2" on the node "ce2". stdout:
INFO[0026] Executed command "ip link add dev vrf-3 type vrf table 3" on the node "ce2". stdout:
INFO[0026] Executed command "ip link set dev vrf-3 up" on the node "ce2". stdout:
INFO[0026] Executed command "ip link set dev eth3 master vrf-3" on the node "ce2". stdout:
INFO[0026] Executed command "ip addr add 192.168.0.21/24 dev eth1" on the node "ce2". stdout:
INFO[0026] Executed command "ip addr add 192.168.0.22/24 dev eth2" on the node "ce2". stdout:
INFO[0026] Executed command "ip addr add 192.168.0.23/24 dev eth3" on the node "ce2". stdout:
+---+-----------------------+--------------+------------------------------+-------+---------+----------------+----------------------+
| # |         Name          | Container ID |            Image             | Kind  |  State  |  IPv4 Address  |     IPv6 Address     |
+---+-----------------------+--------------+------------------------------+-------+---------+----------------+----------------------+
| 1 | clab-evpn-mh01-ce1    | 11d8ad808671 | akpinar/alpine:latest        | linux | running | 172.20.20.2/24 | 2001:172:20:20::2/64 |
| 2 | clab-evpn-mh01-ce2    | f563402d339f | akpinar/alpine:latest        | linux | running | 172.20.20.7/24 | 2001:172:20:20::7/64 |
| 3 | clab-evpn-mh01-leaf1  | dfcf20665a6a | ghcr.io/nokia/srlinux:23.3.1 | srl   | running | 172.20.20.4/24 | 2001:172:20:20::4/64 |
| 4 | clab-evpn-mh01-leaf2  | fee169425f04 | ghcr.io/nokia/srlinux:23.3.1 | srl   | running | 172.20.20.6/24 | 2001:172:20:20::6/64 |
| 5 | clab-evpn-mh01-leaf3  | 115bbac271c9 | ghcr.io/nokia/srlinux:23.3.1 | srl   | running | 172.20.20.5/24 | 2001:172:20:20::5/64 |
| 6 | clab-evpn-mh01-spine1 | d825b06fe483 | ghcr.io/nokia/srlinux:23.3.1 | srl   | running | 172.20.20.3/24 | 2001:172:20:20::3/64 |
+---+-----------------------+--------------+------------------------------+-------+---------+----------------+----------------------+
```

A few seconds later containerlab finishes the deployment with providing a summary table that outlines connection details of the deployed nodes. In the "Name" column we have the names of the deployed containers and those names can be used to reach the nodes, for example to connect to the SSH of `leaf1`:

```bash
# default credentials admin:NokiaSrl1!
ssh admin@clab-evpn01-leaf1
```

To connect Alpine Linux (CEs):

=== "srv1"
    ```
    docker exec -it clab-evpn-mh01-ce1 bash
    ```
=== "srv2"
    ```
    docker exec -it clab-evpn-mh01-ce1 bash
    ```


## EVPN Multi-homing Terminology

Before diving into the hands-on, let's see some terms that would help us better understand the configurations.

+ **Ethernet Segment (ES):** Defines the CE links connected to multiple PEs. An ES is configured in all PEs that a CE is connected and has a unique identifier (ESI) advertised via EVPN.

<div class="mxgraph" style="max-width:100%;border:1px solid transparent;margin:0 auto; display:block;" data-mxgraph="{&quot;page&quot;:1,&quot;zoom&quot;:2,&quot;highlight&quot;:&quot;#0000ff&quot;,&quot;nav&quot;:true,&quot;check-visible-state&quot;:true,&quot;resize&quot;:true,&quot;url&quot;:&quot;https://raw.githubusercontent.com/srl-labs/learn-srlinux/diagrams/evpn-mh.drawio&quot;}"></div>

+ **Multi-homing Modes:** The standard defines two modes: single-active and all-active. Single-active mode has only one active link, while all-active mode uses all links and provides load balancing. This tutorial covers an example of all-active multi-homing.

<div class="mxgraph" style="max-width:100%;border:1px solid transparent;margin:0 auto; display:block;" data-mxgraph="{&quot;page&quot;:2,&quot;zoom&quot;:2,&quot;highlight&quot;:&quot;#0000ff&quot;,&quot;nav&quot;:true,&quot;check-visible-state&quot;:true,&quot;resize&quot;:true,&quot;url&quot;:&quot;https://raw.githubusercontent.com/srl-labs/learn-srlinux/diagrams/evpn-mh.drawio&quot;}"></div>

+ **Link Aggregation Group (LAG):** A LAG is required for all-active but optional for single-active multi-homing.

+ **MAC-VRF:** It is the L2 network-instance, basically a broadcast domain in SR Linux. Interface(s) or LAG must be attached to a MAC-VRF for L2 multi-homing.

The following procedures are essential to EVPN multi-homing but they're not one of the typical configuration items;

+ **Designated Forwarder (DF):** The leaf that is elected to forward BUM traffic. The election is based on the route-type 4 (RT4) exchange, known as the ES routes of EVPN.
+ **Split-horizon (Local bias):** A mechanism to avoid looping the BUM traffic received from the CE back to itself by a peer PE. Local bias is used for all-active and based on RT4 exchange. 
+ **Aliasing:** For remote PEs that are not part of ES to load-balance traffic to the multi-homed CE. RT1 (Auto-discovery) is advertised for aliasing.

EVPN route types 1 and 4 are used to implement the multi-homing procedures. You can check [this](https://documentation.nokia.com/srlinux/23-3/books/evpn-vxlan/evpn-vxlan-tunnels-layer-2.html?hl=designated%2Cforwarder#evpn-l2-multi-hom-procedures) out for more about EVPN multi-homing procedures and route-types.

## SR Linux Multi-Homing Configurations

The following items must be configured in all ES peers(PEs) that provide multi-homing to a CE. It's leaf1 and leaf2 in this tutorial.

+ A LAG and member interfaces
+ Ethernet segment
+ MAC-VRF interface association

Remember that the lab is pre-configured with [fabric underlay][fabric-underlay], [EVPN][evpn], and a [MAC-VRF][mac-vrf] for CE-to-CE L2 communication.

[fabric-underlay]: https://learn.srlinux.dev/tutorials/l2evpn/fabric/
[evpn]: https://learn.srlinux.dev/tutorials/l2evpn/evpn/
[mac-vrf]: https://learn.srlinux.dev/tutorials/l2evpn/evpn/#mac-vrf

### LAG Configuration

LAG is required for the all-active mode but can be skipped for single-active cases.

In this example, a LAG is created for an all-active multi-homing mode. The target setup between a multihomed CE and PEs is shown below.

**IMAGE LAG HERE**

The configuration snippet below shows a LAG with a subinterface and its LACP settings. 
>The same configurations can be copied to both leaf1 and leaf2.

```
enter candidate
    /interface lag1 {
        admin-state enable
        vlan-tagging true
        subinterface 0 {
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
commit now
```

The lag1 is created with the `vlan-tagging` enabled so that this LAG can have multiple subinterfaces with different VLAN tags. In this way, each subinterface can be attached to a different MAC-VRF. The subinterface 0 is created here with `untagged` (tag0) encapsulation.

The `lag type` can be LACP or static. Here it is configured as LACP, so its parameters must match in all nodes, leaf1 and leaf2 in this example.

And associate the physical interface(s) with the LAG to complete this part. 

```
enter candidate
    /interface ethernet-1/1 {
        admin-state enable
        ethernet {
            aggregate-id lag1
        }
    }
commit now
```

All PEs that offer a multi-homing to a CE must be configured similarly with the lag and interface configurations.

### Ethernet Segment Configuration

In SR Linux, the `ethernet-segments` are configured under [ system network-instance protocols evpn ] context.
>The same configuration can be copied to both leaf1 and leaf2.
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
commit now
```

An `ethernet-segment` is created with a name (ES-1) under the BGP-instance 1. The ES identifier (`esi`) and the `multi-homing-mode` must match in all leaf routers. At last, we associate the `lag1` interface with the ES-1.

Besides the ethernet segments, the `bgp-vpn` with `bgp-instance 1` is configured to get the ES routes' BGP information (RT/RD).

### MAC-VRF Interface Configuration

Typically, an L2 multi-homed LAG subinterface needs to be associated with a MAC-VRF.
>The same configuration can be copied to both leaf1 and leaf2.

```
enter candidate
    /network-instance mac-vrf-1 {
        interface lag1.0 {
        }
    }
commit now
```

Also, to enable load-balancing to all-active multi-homing segments, set ecmp to the expected number of leaf (PE) that serves the CE, 2 in this example.
>An ethernet segment can be distrubuted up to 4 PE.

```
enter candidate
    /network-instance mac-vrf-1 {
        protocols {
        bgp-evpn {
            bgp-instance 1 {
                ecmp 2
            }
        }
    }
commit now
```

The whole MAC-VRF with VXLAN configuration is covered [here](https://learn.srlinux.dev/tutorials/l2evpn/evpn/#mac-vrf).

With this, an all-active EVPN-MH configuration is completed.

Let's see the configuration example on the CE side.

## CE (Alpine Linux) Configuration

The ce1 has multi-homed bond0 with slave interfaces eth1 and eth2. Similar to SR Linux part, it's configured with LACP (802.3ad). 

The single homed ce1 has multiple interfaces to a single PE (leaf3). Those interfaces are placed in different VRFs so that ce2 can simulate multiple remote endpoints.
This is primarily to get better enthropy for load-balancing so that you can observe ce1 using both active links.

> See the containerlab topology file for the CE configurations.

## Verifying the Multi-homing

On the PE side, the lag1 was created in both leaf1 and leaf2. Let's see them with these show commands:

```
A:leaf1# show interface lag1
==========================================================================================================================================
lag1 is up, speed None, type None
  lag1.0 is up
    Network-instance: mac-vrf-1
    Encapsulation   : null
    Type            : bridged
------------------------------------------------------------------------------------------------------------------------------------------
==========================================================================================================================================
--{ running }--[  ]--
A:leaf1# show lag lag1 lacp-state
------------------------------------------------------------------------------------------------------------------------------------------
LACP State for lag1
------------------------------------------------------------------------------------------------------------------------------------------
Lag Id         : lag1
Interval       : SLOW
Mode           : ACTIVE
System Id      : 00:00:00:00:00:11
System Priority: 11
+--------------+-----------+-----------+-----------+-----------+-----------+-----------+-----------+-----------+-----------+-----------+
|   Members    |   Oper    | Activity  |  Timeout  |   State   | System Id | Oper key  |  Partner  |  Partner  |  Port No  |  Partner  |
|              |   state   |           |           |           |           |           |    Id     |    Key    |           |  Port No  |
+==============+===========+===========+===========+===========+===========+===========+===========+===========+===========+===========+
| ethernet-1/1 | up        | ACTIVE    | LONG      | IN_SYNC/T | 00:00:00: | 11        | 00:C1:AB: | 15        | 1         | 1         |
|              |           |           |           | rue/True/ | 00:00:11  |           | 00:00:11  |           |           |           |
|              |           |           |           | True      |           |           |           |           |           |           |
+--------------+-----------+-----------+-----------+-----------+-----------+-----------+-----------+-----------+-----------+-----------+
------------------------------------------------------------------------------------------------------------------------------------------
```

The LAG status, network-instance is seen in the `show interface lag1` output. The `show lag lag1 lacp-state` shows the LACP parameters as well as the member ports' state information. The outputs are expected to be the same in leaf1 and leaf2 except from the 'Partner Port No' which is unique per peer.

To see the ES details:

```
A:leaf1# show system network-instance ethernet-segments ES-1 detail
===============================================================================================================
Ethernet Segment
===============================================================================================================
Name                 : ES-1
Admin State          : enable              Oper State        : up
ESI                  : 01:11:11:11:11:11:11:00:00:01
Multi-homing         : all-active          Oper Multi-homing : all-active
Interface            : lag1
Next Hop             : N/A
EVI                  : N/A
ES Activation Timer  : None
DF Election          : default             Oper DF Election  : default

Last Change          : 2023-08-16T14:53:30.270Z
===============================================================================================================
MAC-VRF   Actv Timer Rem   DF
ES-1      0                Yes
---------------------------------------------------------------------------------------------------------------
DF Candidates
---------------------------------------------------------------------------------------------------------------
Network-instance       ES Peers
mac-vrf-1              10.0.0.1
mac-vrf-1              10.0.0.2 (DF)
===============================================================================================================
```
> Verify the same in leaf2.

The configured ES parameters are shown here as well as the ES peers and elected DF.

At this point, let's send some CE-to-CE traffic to see if multi-homing works. Also for SR Linux fabric to learn some MACs. 

Connect to the ce1 and send packets to the remote IP addresses:

=== "nmap"
    ```
    bash-5.0# nmap -PE 192.168.0.21-23
    Starting Nmap 7.80 ( https://nmap.org ) at 2023-08-17 13:52 UTC
    Nmap scan report for 192.168.0.21
    Host is up (0.0027s latency).
    All 1000 scanned ports on 192.168.0.21 are closed
    MAC Address: 00:C1:AB:00:00:21 (Unknown)
    
    Nmap scan report for 192.168.0.22
    Host is up (0.0028s latency).
    All 1000 scanned ports on 192.168.0.22 are closed
    MAC Address: 00:C1:AB:00:00:22 (Unknown)
    
    Nmap scan report for 192.168.0.23
    Host is up (0.0024s latency).
    All 1000 scanned ports on 192.168.0.23 are closed
    MAC Address: 00:C1:AB:00:00:23 (Unknown)
    
    Nmap done: 3 IP addresses (3 hosts up) scanned in 12.80 seconds
    ```
=== "eth1 tcpdump"
    ```
    bash-5.0# tcpdump -ni eth1
    tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
    listening on eth1, link-type EN10MB (Ethernet), capture size 262144 bytes
    13:58:07.487592 LACPv1, length 110
    13:58:10.807635 ARP, Request who-has 192.168.0.22 tell 192.168.0.11, length 28
    13:58:10.807677 ARP, Request who-has 192.168.0.23 tell 192.168.0.11, length 28
    13:58:10.807687 ARP, Request who-has 192.168.0.21 tell 192.168.0.11, length 28
    13:58:10.810885 ARP, Reply 192.168.0.22 is-at 00:c1:ab:00:00:22, length 28
    13:58:10.810990 ARP, Reply 192.168.0.23 is-at 00:c1:ab:00:00:23, length 28
    13:58:10.857873 IP 192.168.0.11.49747 > 192.168.0.23.445: Flags [S], seq 2625981132, win 1024, options [mss 1460], length 0
    13:58:10.857902 IP 192.168.0.11.49747 > 192.168.0.21.445: Flags [S], seq 2625981132, win 1024, options [mss 1460], length 0
    13:58:10.857936 IP 192.168.0.11.49747 > 192.168.0.23.23: Flags [S], seq 2625981132, win 1024, options [mss 1460], length 0
    13:58:10.857949 IP 192.168.0.11.49747 > 192.168.0.21.23: Flags [S], seq 2625981132, win 1024, options [mss 1460], length 0
    13:58:10.857975 IP 192.168.0.11.49747 > 192.168.0.23.3389: Flags [S], seq 2625981132, win 1024, options [mss 1460], length 0
    13:58:10.857989 IP 192.168.0.11.49747 > 192.168.0.21.3389: Flags [S], seq 2625981132, win 1024, options [mss 1460], length 0
    13:58:10.860176 IP 192.168.0.22.23 > 192.168.0.11.49747: Flags [R.], seq 0, ack 2625981133, win 0, length 0
    13:58:10.860265 IP 192.168.0.22.443 > 192.168.0.11.49747: Flags [R.], seq 0, ack 2625981133, win 0, length 0
    13:58:10.862538 IP 192.168.0.11.49747 > 192.168.0.23.443: Flags [S], seq 2625981132, win 1024, options [mss 1460], length 0
    13:58:10.862585 IP 192.168.0.11.49747 > 192.168.0.21.443: Flags [S], seq 2625981132, win 1024, options [mss 1460], length 0
    13:58:10.862647 IP 192.168.0.11.49747 > 192.168.0.23.199: Flags [S], seq 2625981132, win 1024, options [mss 1460], length 0
    13:58:10.862696 IP 192.168.0.11.49747 > 192.168.0.21.199: Flags [S], seq 2625981132, win 1024, options [mss 1460], length 0
    13:58:10.862795 IP 192.168.0.11.49747 > 192.168.0.23.111: Flags [S], seq 2625981132, win 1024, options [mss 1460], length 0
    ```
=== "eth2 tcpdump"
    ```
    bash-5.0# tcpdump -ni eth2
    tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
    listening on eth2, link-type EN10MB (Ethernet), capture size 262144 bytes
    13:58:10.810595 ARP, Reply 192.168.0.21 is-at 00:c1:ab:00:00:21, length 28
    13:58:10.857768 IP 192.168.0.11.49747 > 192.168.0.22.445: Flags [S], seq 2625981132, win 1024, options [mss 1460], length 0
    13:58:10.857916 IP 192.168.0.11.49747 > 192.168.0.22.23: Flags [S], seq 2625981132, win 1024, options [mss 1460], length 0
    13:58:10.857962 IP 192.168.0.11.49747 > 192.168.0.22.3389: Flags [S], seq 2625981132, win 1024, options [mss 1460], length 0
    13:58:10.858002 IP 192.168.0.11.49747 > 192.168.0.22.443: Flags [S], seq 2625981132, win 1024, options [mss 1460], length 0
    13:58:10.859723 IP 192.168.0.22.445 > 192.168.0.11.49747: Flags [R.], seq 0, ack 2625981133, win 0, length 0
    13:58:10.859846 IP 192.168.0.22.3389 > 192.168.0.11.49747: Flags [R.], seq 0, ack 2625981133, win 0, length 0
    13:58:10.862474 IP 192.168.0.11.49747 > 192.168.0.22.199: Flags [S], seq 2625981132, win 1024, options [mss 1460], length 0
    13:58:10.862478 IP 192.168.0.23.445 > 192.168.0.11.49747: Flags [R.], seq 0, ack 2625981133, win 0, length 0
    13:58:10.862592 IP 192.168.0.21.445 > 192.168.0.11.49747: Flags [R.], seq 0, ack 2625981133, win 0, length 0
    13:58:10.862620 IP 192.168.0.11.49747 > 192.168.0.22.111: Flags [S], seq 2625981132, win 1024, options [mss 1460], length 0
    13:58:10.862653 IP 192.168.0.23.3389 > 192.168.0.11.49747: Flags [R.], seq 0, ack 2625981133, win 0, length 0
    13:58:10.862727 IP 192.168.0.11.49747 > 192.168.0.22.143: Flags [S], seq 2625981132, win 1024, options [mss 1460], length 0
    13:58:10.862772 IP 192.168.0.21.3389 > 192.168.0.11.49747: Flags [R.], seq 0, ack 2625981133, win 0, length 0
    13:58:10.866605 IP 192.168.0.11.49747 > 192.168.0.22.80: Flags [S], seq 2625981132, win 1024, options [mss 1460], length 0
    ```

Check the tcpdump of eth1 and eth1 while sending traffic with nmap and see the traffic is balanced both for incoming and outgoing packets.
> Outgoing ARP packets may not be balanced since the load balancing is layer2 by default.

Now, we must have triggered some EVPN routes in the fabric.

Remember the route-types mentioned earlier, let's check what leaf1 and leaf2 (ES peers) advertises:

=== "leaf1"
    ```
    A:leaf1# show network-instance default protocols bgp neighbor 10.0.0.2 advertised-routes evpn
    ---------------------------------------------------------------------------------------------------------------------------------------------------    -----------
    Peer        : 10.0.0.2, remote AS: 100, local AS: 100
    Type        : static
    Description : None
    Group       : iBGP-overlay
    ---------------------------------------------------------------------------------------------------------------------------------------------------    -----------
    Origin codes: i=IGP, e=EGP, ?=incomplete
    ---------------------------------------------------------------------------------------------------------------------------------------------------    -----------
    ---------------------------------------------------------------------------------------------------------------------------------------------------    -----------
    Type 1 Ethernet Auto-Discovery Routes
    +------------------------+--------------------------------+------------+------------------------+------------------------+---------    +------------------------+
    |  Route-distinguisher   |              ESI               |   Tag-ID   |        Next-Hop        |          MED           | LocPref |              Path          |
    +========================+================================+============+========================+========================+=========    +========================+
    | 10.0.0.1:111           | 01:11:11:11:11:11:11:00:00:01  | 0          | 10.0.0.1               | -                      |     100     |                        |
    | 10.0.0.1:111           | 01:11:11:11:11:11:11:00:00:01  | 4294967295 | 10.0.0.1               | -                      |     100     |                        |
    +------------------------+--------------------------------+------------+------------------------+------------------------+---------    +------------------------+
    Type 2 MAC-IP Advertisement Routes
    +---------------------+------------+-------------------+---------------------+---------------------+---------------------+---------    +---------------------+
    | Route-distinguisher |   Tag-ID   |    MAC-address    |     IP-address      |      Next-Hop       |         MED         | LocPref |            Path         |
    +=====================+============+===================+=====================+=====================+=====================+=========    +=====================+
    | 10.0.0.1:111        | 0          | 00:C1:AB:00:00:11 | 0.0.0.0             | 10.0.0.1            | -                   |     100     |                     |
    +---------------------+------------+-------------------+---------------------+---------------------+---------------------+---------    +---------------------+
    ---------------------------------------------------------------------------------------------------------------------------------------------------    -----------
    Type 3 Inclusive Multicast Ethernet Tag Routes
    +---------------------------+------------+---------------------+---------------------------+---------------------------+---------    +---------------------------+
    |    Route-distinguisher    |   Tag-ID   |    Originator-IP    |         Next-Hop          |            MED            | LocPref |               Path            |
    +===========================+============+=====================+===========================+===========================+=========    +===========================+
    | 10.0.0.1:111              | 0          | 10.0.0.1            | 10.0.0.1                  | -                         |     100     |                           |
    +---------------------------+------------+---------------------+---------------------------+---------------------------+---------    +---------------------------+
    ---------------------------------------------------------------------------------------------------------------------------------------------------    -----------
    Type 4 Ethernet Segment Routes
    +---------------------+--------------------------------+---------------------+---------------------+---------------------+---------    +---------------------+
    | Route-distinguisher |              ESI               |   Originating-IP    |      Next-Hop       |         MED         | LocPref |            Path         |
    +=====================+================================+=====================+=====================+=====================+=========    +=====================+
    | 10.0.0.1:0          | 01:11:11:11:11:11:11:00:00:01  | 10.0.0.1            | 10.0.0.1            | -                   |     100     |                     |
    +---------------------+--------------------------------+---------------------+---------------------+---------------------+---------    +---------------------+
    ---------------------------------------------------------------------------------------------------------------------------------------------------    -----------
    ---------------------------------------------------------------------------------------------------------------------------------------------------    -----------
    2 advertised Ethernet Auto-Discovery routes
    1 advertised MAC-IP Advertisement routes
    1 advertised Inclusive Multicast Ethernet Tag routes
    1 advertised Ethernet Segment routes
    0 advertised IP Prefix routes
    ---------------------------------------------------------------------------------------------------------------------------------------------------    -----------
    ```
=== "leaf2"
    ```
    A:leaf2# show network-instance default protocols bgp neighbor 10.0.0.1 advertised-routes evpn
    ---------------------------------------------------------------------------------------------------------------------------------------------------    --
    Peer        : 10.0.0.1, remote AS: 100, local AS: 100
    Type        : static
    Description : None
    Group       : iBGP-overlay
    ---------------------------------------------------------------------------------------------------------------------------------------------------    --
    Origin codes: i=IGP, e=EGP, ?=incomplete
    ---------------------------------------------------------------------------------------------------------------------------------------------------    --
    ---------------------------------------------------------------------------------------------------------------------------------------------------    --
    Type 1 Ethernet Auto-Discovery Routes
    +----------------------+--------------------------------+------------+----------------------+----------------------+---------    +----------------------+
    | Route-distinguisher  |              ESI               |   Tag-ID   |       Next-Hop       |         MED          | LocPref |             Path         |
    +======================+================================+============+======================+======================+=========    +======================+
    | 10.0.0.2:111         | 01:11:11:11:11:11:11:00:00:01  | 0          | 10.0.0.2             | -                    |     100     |                      |
    | 10.0.0.2:111         | 01:11:11:11:11:11:11:00:00:01  | 4294967295 | 10.0.0.2             | -                    |     100     |                      |
    +----------------------+--------------------------------+------------+----------------------+----------------------+---------    +----------------------+
    Type 2 MAC-IP Advertisement Routes
    +--------------------+------------+-------------------+--------------------+--------------------+--------------------+---------    +--------------------+
    |       Route-       |   Tag-ID   |    MAC-address    |     IP-address     |      Next-Hop      |        MED         | LocPref |            Path        |
    |       distinguisher    |            |                   |                    |                    |                    |         |                    |
    +====================+============+===================+====================+====================+====================+=========    +====================+
    | 10.0.0.2:111       | 0          | 00:C1:AB:00:00:11 | 0.0.0.0            | 10.0.0.2           | -                  |     100     |                    |
    +--------------------+------------+-------------------+--------------------+--------------------+--------------------+---------    +--------------------+
    ---------------------------------------------------------------------------------------------------------------------------------------------------    --
    Type 3 Inclusive Multicast Ethernet Tag Routes
    +------------------------+------------+---------------------+------------------------+------------------------+---------+------------------------+
    |  Route-distinguisher   |   Tag-ID   |    Originator-IP    |        Next-Hop        |          MED           | LocPref |          Path          |
    +========================+============+=====================+========================+========================+=========+========================+
    | 10.0.0.2:111           | 0          | 10.0.0.2            | 10.0.0.2               | -                      | 100     |                        |
    +------------------------+------------+---------------------+------------------------+------------------------+---------+------------------------+
    ---------------------------------------------------------------------------------------------------------------------------------------------------    --
    Type 4 Ethernet Segment Routes
    +--------------------+--------------------------------+--------------------+--------------------+--------------------+---------    +--------------------+
    |       Route-       |              ESI               |   Originating-IP   |      Next-Hop      |        MED         | LocPref |            Path        |
    |       distinguisher    |                                |                    |                    |                    |         |                    |
    +====================+================================+====================+====================+====================+=========    +====================+
    | 10.0.0.2:0         | 01:11:11:11:11:11:11:00:00:01  | 10.0.0.2           | 10.0.0.2           | -                  |     100     |                    |
    +--------------------+--------------------------------+--------------------+--------------------+--------------------+---------    +--------------------+
    ---------------------------------------------------------------------------------------------------------------------------------------------------    --
    ---------------------------------------------------------------------------------------------------------------------------------------------------    --
    2 advertised Ethernet Auto-Discovery routes
    1 advertised MAC-IP Advertisement routes
    1 advertised Inclusive Multicast Ethernet Tag routes
    1 advertised Ethernet Segment routes
    0 advertised IP Prefix routes
    ---------------------------------------------------------------------------------------------------------------------------------------------------    --
    ```
RT1, RT3 and RT4 routes are triggered by the configuration (ES and MAC-VRF) while RT2 (MAC-IP Advertisements) routes only appear when a MAC is learnt or statically configured. 

Among those, RT4 is known as ES routes which imported by ES peers for DF election and local biasing (split-horizon). It is only advertised/received by leaf1 and leaf2 here. 

RT1 advertises ESIs as well, basically for two reasons(that's why two entries per ESI); 
+ Aliasing to provide load-balancing (0) 
+ Mass withdrawal for fast convergence (4294967295)

Let's see what leaf2 and leaf3 gets in their BGP EVPN route table:

=== "leaf2"
    ```
    A:leaf2# show network-instance default protocols bgp routes evpn route-type summary
    ------------------------------------------------------------------------------------------------------------------------------------------------------------------
    Show report for the BGP route table of network-instance "default"
    ------------------------------------------------------------------------------------------------------------------------------------------------------------------
    Status codes: u=used, *=valid, >=best, x=stale
    Origin codes: i=IGP, e=EGP, ?=incomplete
    ------------------------------------------------------------------------------------------------------------------------------------------------------------------
    BGP Router ID: 10.0.0.2      AS: 102      Local AS: 102
    ------------------------------------------------------------------------------------------------------------------------------------------------------------------
    ------------------------------------------------------------------------------------------------------------------------------------------------------------------
    Type 1 Ethernet Auto-Discovery Routes
    +--------+-------------------------+--------------------------------+------------+-------------------------+-------------------------+-------------------------+
    | Status |   Route-distinguisher   |              ESI               |   Tag-ID   |        neighbor         |        Next-hop         |           VNI           |
    +========+=========================+================================+============+=========================+=========================+=========================+
    | u*>    | 10.0.0.1:111            | 01:11:11:11:11:11:11:00:00:01  | 0          | 10.0.0.1                | 10.0.0.1                | 1                       |
    | u*>    | 10.0.0.1:111            | 01:11:11:11:11:11:11:00:00:01  | 4294967295 | 10.0.0.1                | 10.0.0.1                | -                       |
    +--------+-------------------------+--------------------------------+------------+-------------------------+-------------------------+-------------------------+
    Type 2 MAC-IP Advertisement Routes
    +-------+--------------+-----------+-----------------+--------------+--------------+--------------+--------------+-----------------------------+--------------+
    | Statu | Route-distin |  Tag-ID   |   MAC-address   |  IP-address  |   neighbor   |   Next-Hop   |     VNI      |             ESI             | MAC Mobility |
    |   s   |   guisher    |           |                 |              |              |              |              |                             |              |
    +=======+==============+===========+=================+==============+==============+==============+==============+=============================+==============+
    | u*>   | 10.0.0.1:111 | 0         | 00:C1:AB:00:00: | 0.0.0.0      | 10.0.0.1     | 10.0.0.1     | 1            | 01:11:11:11:11:11:11:00:00: | -            |
    |       |              |           | 11              |              |              |              |              | 01                          |              |
    | u*>   | 10.0.0.3:111 | 0         | 00:C1:AB:00:00: | 0.0.0.0      | 10.0.0.3     | 10.0.0.3     | 1            | 00:00:00:00:00:00:00:00:00: | -            |
    |       |              |           | 21              |              |              |              |              | 00                          |              |
    | u*>   | 10.0.0.3:111 | 0         | 00:C1:AB:00:00: | 0.0.0.0      | 10.0.0.3     | 10.0.0.3     | 1            | 00:00:00:00:00:00:00:00:00: | -            |
    |       |              |           | 23              |              |              |              |              | 00                          |              |
    +-------+--------------+-----------+-----------------+--------------+--------------+--------------+--------------+-----------------------------+--------------+
    ------------------------------------------------------------------------------------------------------------------------------------------------------------------
    Type 3 Inclusive Multicast Ethernet Tag Routes
    +--------+--------------------------------------+------------+---------------------+--------------------------------------+--------------------------------------+
    | Status |         Route-distinguisher          |   Tag-ID   |    Originator-IP    |               neighbor               |               Next-Hop               |
    +========+======================================+============+=====================+======================================+======================================+
    | u*>    | 10.0.0.1:111                         | 0          | 10.0.0.1            | 10.0.0.1                             | 10.0.0.1                             |
    | u*>    | 10.0.0.3:111                         | 0          | 10.0.0.3            | 10.0.0.3                             | 10.0.0.3                             |
    +--------+--------------------------------------+------------+---------------------+--------------------------------------+--------------------------------------+
    ------------------------------------------------------------------------------------------------------------------------------------------------------------------
    Type 4 Ethernet Segment Routes
    +--------+----------------------------+--------------------------------+----------------------------+----------------------------+----------------------------+
    | Status |    Route-distinguisher     |              ESI               |     originating-router     |          neighbor          |          Next-Hop          |
    +========+============================+================================+============================+============================+============================+
    | u*>    | 10.0.0.1:0                 | 01:11:11:11:11:11:11:00:00:01  | 10.0.0.1                   | 10.0.0.1                   | 10.0.0.1                   |
    +--------+----------------------------+--------------------------------+----------------------------+----------------------------+----------------------------+
    ------------------------------------------------------------------------------------------------------------------------------------------------------------------
    2 Ethernet Auto-Discovery routes 2 used, 2 valid
    3 MAC-IP Advertisement routes 3 used, 3 valid
    2 Inclusive Multicast Ethernet Tag routes 2 used, 2 valid
    1 Ethernet Segment routes 1 used, 1 valid
    0 IP Prefix routes 0 used, 0 valid
    ------------------------------------------------------------------------------------------------------------------------------------------------------------------
    ```
=== "leaf3"
    ```
    A:leaf3# show network-instance default protocols bgp routes evpn route-type summary
    -------------------------------------------------------------------------------------------------------------------------------------------------------------------
    Show report for the BGP route table of network-instance "default"
    -------------------------------------------------------------------------------------------------------------------------------------------------------------------
    Status codes: u=used, *=valid, >=best, x=stale
    Origin codes: i=IGP, e=EGP, ?=incomplete
    -------------------------------------------------------------------------------------------------------------------------------------------------------------------
    BGP Router ID: 10.0.0.3      AS: 103      Local AS: 103
    -------------------------------------------------------------------------------------------------------------------------------------------------------------------
    -------------------------------------------------------------------------------------------------------------------------------------------------------------------
    Type 1 Ethernet Auto-Discovery Routes
    +--------+-------------------------+--------------------------------+------------+-------------------------+-------------------------+-------------------------+
    | Status |   Route-distinguisher   |              ESI               |   Tag-ID   |        neighbor         |        Next-hop         |           VNI           |
    +========+=========================+================================+============+=========================+=========================+=========================+
    | u*>    | 10.0.0.1:111            | 01:11:11:11:11:11:11:00:00:01  | 0          | 10.0.0.1                | 10.0.0.1                | 1                       |
    | u*>    | 10.0.0.1:111            | 01:11:11:11:11:11:11:00:00:01  | 4294967295 | 10.0.0.1                | 10.0.0.1                | -                       |
    | u*>    | 10.0.0.2:111            | 01:11:11:11:11:11:11:00:00:01  | 0          | 10.0.0.2                | 10.0.0.2                | 1                       |
    | u*>    | 10.0.0.2:111            | 01:11:11:11:11:11:11:00:00:01  | 4294967295 | 10.0.0.2                | 10.0.0.2                | -                       |
    +--------+-------------------------+--------------------------------+------------+-------------------------+-------------------------+-------------------------+
    Type 2 MAC-IP Advertisement Routes
    +-------+--------------+-----------+-----------------+--------------+--------------+--------------+--------------+-----------------------------+--------------+
    | Statu | Route-distin |  Tag-ID   |   MAC-address   |  IP-address  |   neighbor   |   Next-Hop   |     VNI      |             ESI             | MAC Mobility |
    |   s   |   guisher    |           |                 |              |              |              |              |                             |              |
    +=======+==============+===========+=================+==============+==============+==============+==============+=============================+==============+
    | u*>   | 10.0.0.1:111 | 0         | 00:C1:AB:00:00: | 0.0.0.0      | 10.0.0.1     | 10.0.0.1     | 1            | 01:11:11:11:11:11:11:00:00: | -            |
    |       |              |           | 11              |              |              |              |              | 01                          |              |
    | u*>   | 10.0.0.2:111 | 0         | 00:C1:AB:00:00: | 0.0.0.0      | 10.0.0.2     | 10.0.0.2     | 1            | 01:11:11:11:11:11:11:00:00: | -            |
    |       |              |           | 11              |              |              |              |              | 01                          |              |
    +-------+--------------+-----------+-----------------+--------------+--------------+--------------+--------------+-----------------------------+--------------+
    -------------------------------------------------------------------------------------------------------------------------------------------------------------------
    Type 3 Inclusive Multicast Ethernet Tag Routes
    +--------+--------------------------------------+------------+---------------------+--------------------------------------+--------------------------------------+
    | Status |         Route-distinguisher          |   Tag-ID   |    Originator-IP    |               neighbor               |               Next-Hop               |
    +========+======================================+============+=====================+======================================+======================================+
    | u*>    | 10.0.0.1:111                         | 0          | 10.0.0.1            | 10.0.0.1                             | 10.0.0.1                             |
    | u*>    | 10.0.0.2:111                         | 0          | 10.0.0.2            | 10.0.0.2                             | 10.0.0.2                             |
    +--------+--------------------------------------+------------+---------------------+--------------------------------------+--------------------------------------+
    -------------------------------------------------------------------------------------------------------------------------------------------------------------------
    4 Ethernet Auto-Discovery routes 4 used, 4 valid
    2 MAC-IP Advertisement routes 2 used, 2 valid
    2 Inclusive Multicast Ethernet Tag routes 2 used, 2 valid
    0 Ethernet Segment routes 0 used, 0 valid
    0 IP Prefix routes 0 used, 0 valid
    -------------------------------------------------------------------------------------------------------------------------------------------------------------------
    ```

The leaf 2, as an ES peer, gets both RT1 and RT4 in its table while leaf3 only imports RT1 as it's a remote PE.

At last, verify the MAC table of the "mac-vrf-1" on Leaf3 which should show the 'esi' instead of an individual destination for the ce1's MAC address.

```
A:leaf3# show network-instance mac-vrf-1 bridge-table mac-table all
-----------------------------------------------------------------------------------------------------------------------------------------------------
Mac-table of network instance mac-vrf-1
-----------------------------------------------------------------------------------------------------------------------------------------------------
+--------------------+---------------------------------------+------------+------------+---------+--------+---------------------------------------+
|      Address       |              Destination              | Dest Index |    Type    | Active  | Aging  |              Last Update              |
+====================+=======================================+============+============+=========+========+=======================================+
| 00:C1:AB:00:00:11  | vxlan-interface:vxlan1.1              | 7173081172 | evpn       | true    | N/A    | 2023-08-17T10:34:59.000Z              |
|                    | esi:01:11:11:11:11:11:11:00:00:01     | 20         |            |         |        |                                       |
| 00:C1:AB:00:00:21  | ethernet-1/1.0                        | 3          | learnt     | true    | 300    | 2023-08-17T10:34:59.000Z              |
| 00:C1:AB:00:00:22  | ethernet-1/2.0                        | 4          | learnt     | true    | 300    | 2023-08-17T10:35:01.000Z              |
| 00:C1:AB:00:00:23  | ethernet-1/3.0                        | 5          | learnt     | true    | 300    | 2023-08-17T10:35:03.000Z              |
+--------------------+---------------------------------------+------------+------------+---------+--------+---------------------------------------+
Total Irb Macs                 :    0 Total    0 Active
Total Static Macs              :    0 Total    0 Active
Total Duplicate Macs           :    0 Total    0 Active
Total Learnt Macs              :    3 Total    3 Active
Total Evpn Macs                :    1 Total    1 Active
Total Evpn static Macs         :    0 Total    0 Active
Total Irb anycast Macs         :    0 Total    0 Active
Total Proxy Antispoof Macs     :    0 Total    0 Active
Total Reserved Macs            :    0 Total    0 Active
Total Eth-cfm Macs             :    0 Total    0 Active
--{ running }--[  ]--
```

With that, we verified a working L2 EVPN multi-homing on a SR Linux fabric. 

[topology]: https://github.com/srl-labs/learn-srlinux/blob/main/docs/tutorials/evpn-mh/
[configs]: https://github.com/srl-labs/learn-srlinux/blob/main/docs/tutorials/evpn-mh/leaf1.cfg
[path-evpn-mh]: https://github.com/srl-labs/learn-srlinux/blob/main/docs/tutorials/evpn-mh
