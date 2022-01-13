---
title: "OpenStack Networking - Where Is My Packet - introduction"
date: 2022-01-13T12:32:52+01:00
draft: false
---

I have been working with the OpenStack networking project
([Neutron](https://docs.openstack.org/neutron/latest/)) for about 6 years now.
First I was working for the public cloud provider [OVH](https://ovh.com) where
we built OpenStack based public cloud services. I was  part of the team which
built a custom, BGP based backend for Neutron. Later on I joined [Red
Hat](https://redhat.com) in their OpenStack networking team, where   I continue
working  until now.
In both companies I have dealt with OpenStack networking issues which in many
cases are pretty similar to each other.

When I work with our customers or support team, I see that one of the common
problems when there are connectivity issues to/from/between instances, is to
really be able to narrow down where the packets are really dropped. This is
generally hard for people to understand as there are many different variables
which may change packet flows completely. It depends on:

* Neutron backend used, like e.g.:
    * ML2 with Open vSwitch - `ML2/OVS` for short,
    * ML2 with OVN - `ML2/OVN` for short,
    * ML2 with Linuxbridge - `Linuxbridge` for short,
    * some 3rd party backend,
* firewall driver - in the case of `ML2/OVS` there can be 2 different drivers:
    * iptables_hybrid which uses iptables to implement security groups for
      ports,
    * openvswitch - which implements security groups using OpenFlow rules,
* scenario, like e.g.:
    * East-West traffic - traffic between VMs in the same L2 network,
    * North-South traffic - traffic from VM to the external world,
    * trunk ports,
    * distributed (`DVR`) or centralized routers (`L3HA` or `legacy`),
    * Neutron network type:
        * vlan or flat,
        * tunnel networks, like `vxlan`, `GRE` or `geneve`

Each of those variables changes, sometimes fundamentally, the way  virtual
machine's ports are provisioned and plugged to the network and how packets
should go to/from them.

Due to all these considerations, it's really hard for people to know what are
all  the "hops" for the packet inside the compute node where the VM is running
and later e.g. in the networker/controller node and to properly narrow down
where the issue really can be.

To help understand that complex beast which Neutron is, I plan to write in the
next few weeks a series of posts where I will try to explain in detail and step
by step the packet flow in various scenarios and how to troubleshoot
connectivity issues in those scenarios.

I plan to describe:
* E-W and N-S traffic in vxlan tenant network, vlan provider network and
  centralized router,
* E-W and N-S traffic in vxlan tenant network, vlan provider network, and DVR
  router,
* Trunk ports - connectivity between 2 VMs.

I work mostly on the `ML2/OVS` backend thus I will focus my series of posts on
that backend. That's the one which I know the most. It is also probably still
the most popular backend used by the community (even if currently  the "hottest"
backend is `ML2/OVN`).

So if You are interested in that topic, stay tuned and get back here in a few
weeks, when I should  have something for You to read :)
