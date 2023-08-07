# SRL Generic Lab

Welcome to the generic SR Linux lab where you can learn and experiment Nokia's SR Linux network operating system. 

During this lab, you can:
 - Deploy pre-configured SR Linux routers(Leaf-Spine) and workloads(Alpine hosts) with containerlab.
 - Explore the underlay routing with eBGP and a typical BGP EVPN setup with Route-Reflectors.
 - See the IP-VRF and MAC-VRF configurations.
 - Pass traffic between connected hosts.
 - And whatever you want to discover on SR Linux...

**Grading: Beginner**
**Elements: SR Linux, Alpine Linux**

## Topology

The lab consist of 2-Tier CLOS data center fabric topology with four hosts. 

The Leaf and Spine notes are running containerized SR Linux (7220 IXR D3L) NOS and hosts are based on Alpine Linux with some networking tools.

See below the connections.

<p align="center"> <img src="https://github.com/hansthienpondt/SReXperts/assets/17744051/bb4c5076-2375-405d-a2b2-c1f8c1a1cf57" height="600"> </p>

Every node in the data center fabric is configured with eBGP as underlay routing protocol. Also iBGP EVPN for overlay ip-vrf and mac-vrf network instances.

Below network instances and client connections are pre-configured.

<p align="center"> <img src="https://github.com/hansthienpondt/SReXperts/assets/17744051/21f38868-8f27-4bf5-984a-b9a48f794525" height="400"> </p>


## Deploying the lab

Deploy the topology defined in srl-eh.yml with containerlab. 
```
containerlab deploy -t srl-generic.yml
```
Source the 'srl-generic.rc' to access nodes with aliases. (E.g. 'l1' instead of 'ssh admin@clab-srl-generic-leaf1')

When you connect to the hosts, run `./restart-services.sh` script to get IP addresses and routes configured.

## Tools needed  

| Role | Software |
| --- | --- |
| Lab Emulation | [containerlab](https://containerlab.dev/) |

## Credentials & Access

You can access SR Linux nodes and hosts via SSH. The container names listed after the containerlab deployment are added to the hosts file, so you can directly ssh to the name of the container, like "ssh admin@clab-srl-generic-leaf1".

| Node    | SSH (Internal)                         | User/Pass        |
| ------- | -------------------------------------- | ---------------- |
| spine1  | `ssh admin@clab-srl-generic-spine1`    | admin/NokiaSrl1! |
| spine2  | `ssh admin@clab-srl-generic-spine2`    | admin/NokiaSrl1! |
| leaf1   | `ssh admin@clab-srl-generic-leaf1 `    | admin/NokiaSrl1! |
| leaf2   | `ssh admin@clab-srl-generic-leaf2 `    | admin/NokiaSrl1! |
| leaf3   | `ssh admin@clab-srl-generic-leaf3 `    | admin/NokiaSrl1! |
| leaf4   | `ssh admin@clab-srl-generic-leaf4 `    | admin/NokiaSrl1! |
| h1      | `ssh root@clab-srl-generic-h1`         | root/clab        |
| h2      | `ssh root@clab-srl-generic-h2`         | root/clab        |
| h3      | `ssh root@clab-srl-generic-h3`         | root/clab        |
| h4      | `ssh root@clab-srl-generic-h4`         | root/clab        |

Alternatively, source the 'srl-generic.rc' to access nodes with aliases direcly from your VM. (E.g. 'l1' instead of 'ssh admin@clab-srl-generic-leaf1')

If you wish to have direct external access from your machine, use the public IP address of the VM and the external port numbers as per the table below:

| Node    | Direct SSH (Ext.)        | gNMI(Ext.) |
| ------- | ------------------------ | ---------- |
| spine1  | `ssh admin@IP:55001`     | `IP:55301` |
| spine2  | `ssh admin@IP:55002`     | `IP:55302` |
| leaf1   | `ssh admin@IP:55003`     | `IP:55303` |
| leaf2   | `ssh admin@IP:55004`     | `IP:55304` |
| leaf3   | `ssh admin@IP:55005`     | `IP:55305` |
| leaf4   | `ssh admin@IP:55006`     | `IP:55306` |
| h1      | `ssh root@IP:55007`      |            |
| h2      | `ssh root@IP:55008`      |            |
| h3      | `ssh root@IP:55009`      |            |
| h4      | `ssh root@IP:55010`      |            |
 
## Exploring SR Linux 

