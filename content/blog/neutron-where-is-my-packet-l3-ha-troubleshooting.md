---
title: "Neutron Where Is My Packet L3 Ha Troubleshooting"
date: 2022-04-19T11:42:35+02:00
draft: true
---

## Intro

This is yet another post about debugging [OpenStack Neutron](https://docs.openstack.org/neutron/latest/).
In the [previous post](/blog/neutron-where-is-my-packet-2/) I described how
East-West between VMs, VMs and routers, VMs and metadata servers and VMs and
DHCP servers works. I also described North-South connectivity between VM and
"external world". All of that was related to the routers called ``legacy`` in
Neutron. That types of routers are centralized and it is always hosted only on
one Neutron L3 agent.

Neutron supports also 3 different types of routers:
* High Availability routers (``l3_ha``) - which are very similar to the
  ``legacy`` routers but such router is hosted in the same time by more than one
  L3 agent and can quickly failover between nodes if ``primary`` node is down,
* Distributed Virtual Routers (``DVR``) - routers distributed through the
  compute nodes - such routers lives on the compute nodes where there are
  instances connected to it. DNAT traffic (Floating IPs) is distributed and
  is going outside to the external world directly from the compute node. SNAT
  traffic is still centralized and is going through the networker node always,
* there is also combination of the 2 mentioned above, called ``DVR-HA`` - in
  that case, DNAT traffic is distributed across compute nodes and centralized
  SNAT routers are in the HA mode, in the same way how centralized ``l3_ha``
  routers works.

In this post I will focus on the centralized L3 HA routers and I will try to
explain how that HA feature works there, how to investigate if/where router is
active and what to check if something isn't working as expected there. I will
not describe here any traffic from VM to metadata/DHCP or between VMs as that is
exactly the same as for the legacy routers, which was described with details in
the [previous post](/blog/neutron-where-is-my-packet-2/). The only thing which
needs to be kept in mind here is that all that traffic is always going to the
node where router is in the ``primary`` state.

Some more detailed description of the L3 HA configuration can be found in the
[Neutron
docs](https://docs.openstack.org/neutron/pike/admin/deploy-ovs-ha-vrrp.html#deploy-ovs-ha-vrrp)

### Test environment

I will do all of that on the small, virtual environment with 3 nodes deployed
using
[Devstack](https://docs.openstack.org/devstack/latest/guides/multinode-lab.html):
One “all-in-one” node which will run as controller and also as compute node, 2
“compute” nodes which will run only as compute nodes to host instances. Those
"compute" nodes will also be working as the "networker" nodes so they will host
L3 agents.

From the Neutron perspective it looks like below:

```bash
$ openstack network agent list
+--------------------------------------+--------------------+----------------------------+-------------------+-------+-------+---------------------------+
| ID                                   | Agent Type         | Host                       | Availability Zone | Alive | State | Binary                    |
+--------------------------------------+--------------------+----------------------------+-------------------+-------+-------+---------------------------+
| 07fa5b62-593f-44eb-a6b7-a07fddd585df | L3 agent           | devstack-ubuntu-controller | nova              | :-)   | UP    | neutron-l3-agent          |
| 218c4118-c7f6-4f1c-8669-11812d606c24 | L3 agent           | devstack-ubuntu-compute-2  | nova              | :-)   | UP    | neutron-l3-agent          |
| 28f0a978-e234-4842-be47-34eebabf9dd8 | Metadata agent     | devstack-ubuntu-compute-1  | None              | :-)   | UP    | neutron-metadata-agent    |
| 49e1adf4-e4db-478b-bcf0-0361f40757d3 | DHCP agent         | devstack-ubuntu-controller | nova              | :-)   | UP    | neutron-dhcp-agent        |
| 7f3ea902-7267-4c4a-81b0-c9f708759242 | Open vSwitch agent | devstack-ubuntu-compute-2  | None              | :-)   | UP    | neutron-openvswitch-agent |
| bae357c5-90a8-4305-8ebd-3913e63f68f9 | Metadata agent     | devstack-ubuntu-controller | None              | :-)   | UP    | neutron-metadata-agent    |
| be501d8e-6a9e-4a34-93cf-2d5514ff2866 | Open vSwitch agent | devstack-ubuntu-controller | None              | :-)   | UP    | neutron-openvswitch-agent |
| c0bda127-e6cb-44ee-be18-31f4bc62d30c | Metadata agent     | devstack-ubuntu-compute-2  | None              | :-)   | UP    | neutron-metadata-agent    |
| df4b09b7-f171-4f16-9506-81d09fd72fc8 | Open vSwitch agent | devstack-ubuntu-compute-1  | None              | :-)   | UP    | neutron-openvswitch-agent |
| e30a4917-a04c-4445-83de-fbe3ba77de22 | L3 agent           | devstack-ubuntu-compute-1  | nova              | :-)   | UP    | neutron-l3-agent          |
+--------------------------------------+--------------------+----------------------------+-------------------+-------+-------+---------------------------+
```

### Use case scenario

In this post I will describe a case with:
* tunnel (``vxlan``) tenant networks,
* flat provider network, used as external one,
* 2 virtual machines,
* L3 HA router,
* Floating IP attached to one of the instances, the other one will have access
  to the “internet” using SNAT functionality of the router.

All of this is shown on the image below:

![network-topology](/images/debugging-neutron/ha-routing/network-topology.png)

Router is hosted on all 3 nodes. It is in the ``primary`` state on the
``devstack-ubuntu-controller`` node

```bash
$ neutron l3-agent-list-hosting-router router1
neutron CLI is deprecated and will be removed in the Z cycle. Use openstack CLI instead.
+--------------------------------------+----------------------------+----------------+-------+----------+
| id                                   | host                       | admin_state_up | alive | ha_state |
+--------------------------------------+----------------------------+----------------+-------+----------+
| 07fa5b62-593f-44eb-a6b7-a07fddd585df | devstack-ubuntu-controller | True           | :-)   | active   |
| 218c4118-c7f6-4f1c-8669-11812d606c24 | devstack-ubuntu-compute-2  | True           | :-)   | standby  |
| e30a4917-a04c-4445-83de-fbe3ba77de22 | devstack-ubuntu-compute-1  | True           | :-)   | standby  |
+--------------------------------------+----------------------------+----------------+-------+----------+
```

## HA tenant network plugged to the router

## Router on the ``primary`` node

## Router on the ``standby`` node

## Processes running for each HA router

### keepalived
### neutron-keepalived-state-change-monitor
### haproxy (metadata)
### others

### Potential problems with the L3 High Availiability routers
