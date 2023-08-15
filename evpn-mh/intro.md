---
comments: true
tags:
  - evpn
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
EVPN provides multi-homing with the Ethernet Segments (ES). ES simply defines all the links from a multi-homed CE to multiple PEs.

This tutorial will show how to configure multi-homing for a CE connected to multiple PEs in an EVPN-based SR Linux fabric.

The lab consists of a spine and 3 leaf(PEs) routers, and two Alpine hosts(CEs). A multi-homed CE is connected to leaf1 and 3, while another CE is connected to leaf3 for testing purposes.

<div class="mxgraph" style="max-width:100%;border:1px solid transparent;margin:0 auto; display:block;" data-mxgraph="{&quot;page&quot;:0,&quot;zoom&quot;:2,&quot;highlight&quot;:&quot;#0000ff&quot;,&quot;nav&quot;:true,&quot;check-visible-state&quot;:true,&quot;resize&quot;:true,&quot;url&quot;:&quot;https://github.com/aaakpinar/NCE/blob/evpn-mh/evpn-mh/evpn-mh-fabric.svg&quot;}"></div>


![image](https://github.com/aaakpinar/NCE/blob/evpn-mh/evpn-mh/evpn-mh-fabric.svg)



The two servers are connected to the leafs via an L2 interface. Service-wise the servers will appear to be on the same L2 network by means of the deployed EVPN Layer 2 service.

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
