---
title: "OpenStack Networking - Where Is My Packet - tunnel tenant networks, centralized routers"
date: 2022-02-02T10:32:52+01:00
draft: false
---

## Intro

This is the first post on how to track packet flows in OpenStack Neutron
connectivity. I want to focus here on packet flows to/from virtual machines in a
scenario with the `ML2` core plugin and the `openvswitch` mechanism driver. Some
of the demonstrated steps and commands require access to the OpenStack API and
others require root user privileges in the nodes of the OpenStack cluster. For a
detailed description on how Neutron agents work together, please refer to the
upstream [Neutron
Documentation](https://docs.openstack.org/neutron/latest/admin/).

### Test environment

I will do all of that on the small, virtual environment with 3 nodes deployed
using
[Devstack](https://docs.openstack.org/devstack/latest/guides/multinode-lab.html):
One “all-in-one” node which will run as controller and also as compute node, 2
“compute” nodes which will run only as compute nodes to host instances.

From the Neutron perspective it looks like below:

```bash
$ openstack network agent list
+--------------------------------------+--------------------+----------------------------+-------------------+-------+-------+---------------------------+
| ID                                   | Agent Type         | Host                       | Availability Zone | Alive | State | Binary                    |
+--------------------------------------+--------------------+----------------------------+-------------------+-------+-------+---------------------------+
| 0e67dd28-16fc-4a1e-91a4-3d41804dccac | Open vSwitch agent | devstack-ubuntu-compute-1  | None              | :-)   | UP    | neutron-openvswitch-agent |
| 4e7ff3e9-c5bb-42f9-b62b-cab7c2e2c4db | Metadata agent     | devstack-ubuntu-controller | None              | :-)   | UP    | neutron-metadata-agent    |
| 6980700d-e9aa-4079-9cd2-508c10c1018d | L3 agent           | devstack-ubuntu-compute-1  | nova              | :-)   | UP    | neutron-l3-agent          |
| 731fab35-3f14-4e66-a7d3-626f0c1477fc | Metadata agent     | devstack-ubuntu-compute-1  | None              | :-)   | UP    | neutron-metadata-agent    |
| 90da145b-108b-4592-92bd-44a7521b9ce3 | Metadata agent     | devstack-ubuntu-compute-2  | None              | :-)   | UP    | neutron-metadata-agent    |
| a1520b60-8f2d-4602-8c9d-0f1bf7febc19 | L3 agent           | devstack-ubuntu-controller | nova              | :-)   | UP    | neutron-l3-agent          |
| b4b9ae69-4ed6-4722-a5a0-cf9e6f3940fe | L3 agent           | devstack-ubuntu-compute-2  | nova              | :-)   | UP    | neutron-l3-agent          |
| c525aa1c-efd2-46ae-be5d-5649aee93649 | Open vSwitch agent | devstack-ubuntu-compute-2  | None              | :-)   | UP    | neutron-openvswitch-agent |
| dede9375-93d9-4f91-ae8e-dbff88798847 | Open vSwitch agent | devstack-ubuntu-controller | None              | :-)   | UP    | neutron-openvswitch-agent |
| f776580f-34e8-48b9-8891-20a92a3e0218 | DHCP agent         | devstack-ubuntu-controller | nova              | :-)   | UP    | neutron-dhcp-agent        |
+--------------------------------------+--------------------+----------------------------+-------------------+-------+-------+---------------------------+
```

### Use case scenario

In this article I want to describe in details a case with:
* Tunnel (vxlan) tenant network,
* Flat provider network (for vlan network it would be very similar, I will
  highlight later the differences between those two types of the provider
  networks),
* 2 Virtual Machines connected to the tenant network:
    * vm1 running on the `devstack-ubuntu-compute-1` node,
    * vm2 running on the `devstack-ubuntu-compute-2` node,
* Legacy router - I will also show differences between legacy router and
  centralized HA routers,
* Floating IP attached to one of the instances, the other one will have access
  to the “internet” using SNAT functionality of the router.

All of this is shown on the image below:

![network-topology](/images/debugging-neutron/legacy-routing/network-topology.png)


Resources that I used while writing this article are the following:

* provider network, named `public`

```bash
$ openstack network show public
+---------------------------+------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| Field                     | Value                                                                                                                                                                              |
+---------------------------+------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| admin_state_up            | UP                                                                                                                                                                                 |
| availability_zone_hints   |                                                                                                                                                                                    |
| availability_zones        | nova                                                                                                                                                                               |
| created_at                | 2022-01-19T12:48:40Z                                                                                                                                                               |
| description               |                                                                                                                                                                                    |
| dns_domain                | None                                                                                                                                                                               |
| id                        | b12c783e-9439-4967-890a-c87762a3f04b                                                                                                                                               |
| ipv4_address_scope        | None                                                                                                                                                                               |
| ipv6_address_scope        | None                                                                                                                                                                               |
| is_default                | True                                                                                                                                                                               |
| is_vlan_transparent       | None                                                                                                                                                                               |
| location                  | Munch({'cloud': '', 'region_name': 'RegionOne', 'zone': None, 'project': Munch({'id': '761282cf528e487fbfc5e310d92e41bb', 'name': None, 'domain_id': None, 'domain_name': None})}) |
| mtu                       | 1500                                                                                                                                                                               |
| name                      | public                                                                                                                                                                             |
| port_security_enabled     | True                                                                                                                                                                               |
| project_id                | 761282cf528e487fbfc5e310d92e41bb                                                                                                                                                   |
| provider:network_type     | flat                                                                                                                                                                               |
| provider:physical_network | default                                                                                                                                                                            |
| provider:segmentation_id  | None                                                                                                                                                                               |
| qos_policy_id             | None                                                                                                                                                                               |
| revision_number           | 3                                                                                                                                                                                  |
| router:external           | External                                                                                                                                                                           |
| segments                  | None                                                                                                                                                                               |
| shared                    | False                                                                                                                                                                              |
| status                    | ACTIVE                                                                                                                                                                             |
| subnets                   | c9d5090f-d99e-40f5-a249-b2f8c313d984, eeb7856c-73ef-405f-8dc5-e836b46bc73a                                                                                                         |
| tags                      |                                                                                                                                                                                    |
| updated_at                | 2022-01-19T12:48:52Z                                                                                                                                                               |
+---------------------------+------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
```

* tenant, tunnel network - named `private`

```bash
$ openstack network show private
+---------------------------+-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| Field                     | Value                                                                                                                                                                                     |
+---------------------------+-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| admin_state_up            | UP                                                                                                                                                                                        |
| availability_zone_hints   |                                                                                                                                                                                           |
| availability_zones        | nova                                                                                                                                                                                      |
| created_at                | 2022-01-19T12:48:31Z                                                                                                                                                                      |
| description               |                                                                                                                                                                                           |
| dns_domain                | None                                                                                                                                                                                      |
| id                        | 5d90176f-d6e4-461e-94c1-c0f44e7cc52b                                                                                                                                                      |
| ipv4_address_scope        | None                                                                                                                                                                                      |
| ipv6_address_scope        | None                                                                                                                                                                                      |
| is_default                | None                                                                                                                                                                                      |
| is_vlan_transparent       | None                                                                                                                                                                                      |
| location                  | Munch({'cloud': '', 'region_name': 'RegionOne', 'zone': None, 'project': Munch({'id': '97919446b3d64ebaa0b53364feba8b75', 'name': 'demo', 'domain_id': 'default', 'domain_name': None})}) |
| mtu                       | 1450                                                                                                                                                                                      |
| name                      | private                                                                                                                                                                                   |
| port_security_enabled     | True                                                                                                                                                                                      |
| project_id                | 97919446b3d64ebaa0b53364feba8b75                                                                                                                                                          |
| provider:network_type     | vxlan                                                                                                                                                                                     |
| provider:physical_network | None                                                                                                                                                                                      |
| provider:segmentation_id  | 1707                                                                                                                                                                                      |
| qos_policy_id             | None                                                                                                                                                                                      |
| revision_number           | 3                                                                                                                                                                                         |
| router:external           | Internal                                                                                                                                                                                  |
| segments                  | None                                                                                                                                                                                      |
| shared                    | False                                                                                                                                                                                     |
| status                    | ACTIVE                                                                                                                                                                                    |
| subnets                   | b3351636-3c2e-45be-bfe5-491b7de03e1e, e5aa5e0e-cd28-4b0c-8a8d-3af43ff6a8ba                                                                                                                |
| tags                      |                                                                                                                                                                                           |
| updated_at                | 2022-01-19T12:48:34Z                                                                                                                                                                      |
+---------------------------+-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
```

* router - named `router1`

```bash
$ openstack router show router1
+-------------------------+-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| Field                   | Value                                                                                                                                                                                                                                                                             |
+-------------------------+-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| admin_state_up          | UP                                                                                                                                                                                                                                                                                |
| availability_zone_hints |                                                                                                                                                                                                                                                                                   |
| availability_zones      | nova                                                                                                                                                                                                                                                                              |
| created_at              | 2022-01-19T12:48:38Z                                                                                                                                                                                                                                                              |
| description             |                                                                                                                                                                                                                                                                                   |
| distributed             | False                                                                                                                                                                                                                                                                             |
| external_gateway_info   | {"network_id": "b12c783e-9439-4967-890a-c87762a3f04b", "external_fixed_ips": [{"subnet_id": "eeb7856c-73ef-405f-8dc5-e836b46bc73a", "ip_address": "10.10.0.240"}, {"subnet_id": "c9d5090f-d99e-40f5-a249-b2f8c313d984", "ip_address": "2001:db8::2c0"}], "enable_snat": true}     |
| flavor_id               | None                                                                                                                                                                                                                                                                              |
| ha                      | False                                                                                                                                                                                                                                                                             |
| id                      | 50711ca1-1e58-44dc-bd95-a87250d5b4ae                                                                                                                                                                                                                                              |
| interfaces_info         | [{"port_id": "2ff5e542-7418-4041-a2f1-e34cf08ac34b", "ip_address": "fdba:b3c4:c275::1", "subnet_id": "b3351636-3c2e-45be-bfe5-491b7de03e1e"}, {"port_id": "7297041d-55de-405a-8d15-5786188346a3", "ip_address": "10.0.0.1", "subnet_id": "e5aa5e0e-cd28-4b0c-8a8d-3af43ff6a8ba"}] |
| location                | Munch({'cloud': '', 'region_name': 'RegionOne', 'zone': None, 'project': Munch({'id': '97919446b3d64ebaa0b53364feba8b75', 'name': 'demo', 'domain_id': 'default', 'domain_name': None})})                                                                                         |
| name                    | router1                                                                                                                                                                                                                                                                           |
| project_id              | 97919446b3d64ebaa0b53364feba8b75                                                                                                                                                                                                                                                  |
| revision_number         | 6                                                                                                                                                                                                                                                                                 |
| routes                  |                                                                                                                                                                                                                                                                                   |
| status                  | ACTIVE                                                                                                                                                                                                                                                                            |
| tags                    |                                                                                                                                                                                                                                                                                   |
| updated_at              | 2022-01-19T12:48:53Z                                                                                                                                                                                                                                                              |
+-------------------------+-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
```

* virtual machines - named `vm1` and `vm2`

```bash
$ openstack server list
+--------------------------------------+------+--------+---------------------------------------------------------------------+--------------------------+----------+
| ID                                   | Name | Status | Networks                                                            | Image                    | Flavor   |
+--------------------------------------+------+--------+---------------------------------------------------------------------+--------------------------+----------+
| ea6f99f3-7da7-4645-858f-b3e2f2860b53 | vm2  | ACTIVE | private=10.0.0.50, fdba:b3c4:c275:0:f816:3eff:fe4a:97e2             | cirros-0.5.1-x86_64-disk | m1.micro |
| d0c91162-3607-4fcc-b22f-5d04f51bc431 | vm1  | ACTIVE | private=10.0.0.57, 10.10.0.151, fdba:b3c4:c275:0:f816:3eff:fe3d:ff6 | cirros-0.5.1-x86_64-disk | m1.micro |
+--------------------------------------+------+--------+---------------------------------------------------------------------+--------------------------+----------+
```

* Floating IP associated with `vm1`

```bash
$ openstack port list --device-id d0c91162-3607-4fcc-b22f-5d04f51bc431
+--------------------------------------+------+-------------------+----------------------------------------------------------------------------------------------------+--------+
| ID                                   | Name | MAC Address       | Fixed IP Addresses                                                                                 | Status |
+--------------------------------------+------+-------------------+----------------------------------------------------------------------------------------------------+--------+
| 3e399bf2-3185-4ed2-8a6b-1d79b84c9605 |      | fa:16:3e:3d:0f:f6 | ip_address='10.0.0.57', subnet_id='e5aa5e0e-cd28-4b0c-8a8d-3af43ff6a8ba'                           | ACTIVE |
|                                      |      |                   | ip_address='fdba:b3c4:c275:0:f816:3eff:fe3d:ff6', subnet_id='b3351636-3c2e-45be-bfe5-491b7de03e1e' |        |
+--------------------------------------+------+-------------------+----------------------------------------------------------------------------------------------------+--------+

$ openstack floating ip create --port 3e399bf2-3185-4ed2-8a6b-1d79b84c9605 public
+---------------------+-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| Field               | Value                                                                                                                                                                                                                                   |
+---------------------+-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| created_at          | 2022-01-19T14:01:28Z                                                                                                                                                                                                                    |
| description         |                                                                                                                                                                                                                                         |
| dns_domain          | None                                                                                                                                                                                                                                    |
| dns_name            | None                                                                                                                                                                                                                                    |
| fixed_ip_address    | 10.0.0.57                                                                                                                                                                                                                               |
| floating_ip_address | 10.10.0.151                                                                                                                                                                                                                             |
| floating_network_id | b12c783e-9439-4967-890a-c87762a3f04b                                                                                                                                                                                                    |
| id                  | 89824995-e3fc-4ede-a457-1407f1db5104                                                                                                                                                                                                    |
| location            | Munch({'cloud': '', 'region_name': 'RegionOne', 'zone': None, 'project': Munch({'id': '97919446b3d64ebaa0b53364feba8b75', 'name': 'demo', 'domain_id': 'default', 'domain_name': None})})                                               |
| name                | 10.10.0.151                                                                                                                                                                                                                             |
| port_details        | {'name': '', 'network_id': '5d90176f-d6e4-461e-94c1-c0f44e7cc52b', 'mac_address': 'fa:16:3e:3d:0f:f6', 'admin_state_up': True, 'status': 'ACTIVE', 'device_id': 'd0c91162-3607-4fcc-b22f-5d04f51bc431', 'device_owner': 'compute:nova'} |
| port_id             | 3e399bf2-3185-4ed2-8a6b-1d79b84c9605                                                                                                                                                                                                    |
| project_id          | 97919446b3d64ebaa0b53364feba8b75                                                                                                                                                                                                        |
| qos_policy_id       | None                                                                                                                                                                                                                                    |
| revision_number     | 0                                                                                                                                                                                                                                       |
| router_id           | 50711ca1-1e58-44dc-bd95-a87250d5b4ae                                                                                                                                                                                                    |
| status              | DOWN                                                                                                                                                                                                                                    |
| subnet_id           | None                                                                                                                                                                                                                                    |
| tags                | []                                                                                                                                                                                                                                      |
| updated_at          | 2022-01-19T14:01:28Z                                                                                                                                                                                                                    |
+---------------------+-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
```

## Trace packets

First of all, I think it's worth seeing how connectivity of the virtual machine,
DHCP port and router is done in a scenario like this. It is all shown in the
diagram from the Neutron documentation:

![components_connectivity](https://docs.openstack.org/neutron/latest/_images/deploy-ovs-selfservice-compconn1.png)

In this example, the `iptables_hybrid` firewall driver is used in the compute
node. With this driver, an additional Linuxbridge called `qbr` is needed between
the instance and the integration bridge (`br-int`).  The other interface in the
`qbr` bridge is the [veth
pair](https://developers.redhat.com/blog/2018/10/22/introduction-to-linux-interfaces-for-virtual-networking#veth)
which has its second end plugged into the Openvswitch integration bridge
(`br-int`). From that point on wiring the port is the neutron-ovs-agent's job.
Another thing worth mentioning here is the fact that the DHCP service is placed
in the compute node, but that isn't required. DHCP can be done in any node where
the Neutron DHCP agent is running.

### Creating new Virtual Machine - debugging DHCP

Lets start from the beginning and create an instance (virtual machine) connected
to the private, tenant network.
When new virtual machine is spawned, and it is finally started by Nova, the
first thing that needs to be done is to get an IP address from the DHCP server.
Neutron's DHCP agent is responsible to configure the DHCP service for the
virtual networks.  Typically such DHCP agent runs on the controller, or
the networking nodes. In the case of this excercise there was one DHCP agent
running in the `devstack-ubuntu-controller` (all-in-one) node.

To configure the DHCP service for network, the Neutron DHCP agent creates a
network namespace called `qdhcp-<network_uuid>` and spawns a `dnsmasq` process
there:

```bash
$ ip netns
qdhcp-5d90176f-d6e4-461e-94c1-c0f44e7cc52b (id: 0)

$ ps aux | grep dnsmasq
nobody    322219  0.0  0.0  12144  2468 ?        S    sty19   0:00 dnsmasq --no-hosts  --pid-file=/opt/stack/data/neutron/dhcp/5d90176f-d6e4-461e-94c1-c0f44e7cc52b/pid --dhcp-hostsfile=/opt/stack/data/neutron/dhcp/5d90176f-d6e4-461e-94c1-c0f44e7cc52b/host --addn-hosts=/opt/stack/data/neutron/dhcp/5d90176f-d6e4-461e-94c1-c0f44e7cc52b/addn_hosts --dhcp-optsfile=/opt/stack/data/neutron/dhcp/5d90176f-d6e4-461e-94c1-c0f44e7cc52b/opts --dhcp-leasefile=/opt/stack/data/neutron/dhcp/5d90176f-d6e4-461e-94c1-c0f44e7cc52b/leases --dhcp-match=set:ipxe,175 --dhcp-userclass=set:ipxe6,iPXE --local-service --bind-dynamic --dhcp-range=set:subnet-e5aa5e0e-cd28-4b0c-8a8d-3af43ff6a8ba,10.0.0.0,static,255.255.255.192,86400s --dhcp-option-force=option:mtu,1450 --dhcp-lease-max=64 --conf-file=/dev/null --domain=openstacklocal

$ sudo ip netns identify 322219
qdhcp-5d90176f-d6e4-461e-94c1-c0f44e7cc52b
```

The leases configuration files used by the dnsmasq process are usually in the
`/var/lib/neutron/dhcp/<network_uuid>/` but it can be placed e.g. in
`/opt/stack/data/neutron/dhcp/<network_uuid>/` in when Neutron is deployed with
Devstack.
For each port created, there is an entry in the leases file, so virtual machines
can get the IP addresses previously allocated for them by Neutron.

Now, when a VM boots, its operating system send a broadcast DHCP request to the
network. If You look at the picture above, You can see that the plug point to
the network in the compute node is the `tap` port in the `qbr` bridge. So let's
check with tcpdump if the DHCP request is going through that port:

```bash
$ sudo tcpdump -i tap3e399bf2-31 -nle not port 22
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on tap3e399bf2-31, link-type EN10MB (Ethernet), capture size 262144 bytes
15:50:12.815970 fa:16:3e:3d:0f:f6 > ff:ff:ff:ff:ff:ff, ethertype IPv4 (0x0800), length 342: 0.0.0.0.68 > 255.255.255.255.67: BOOTP/DHCP, Request from fa:16:3e:3d:0f:f6, length 300
15:50:12.816561 fa:16:3e:65:83:63 > fa:16:3e:3d:0f:f6, ethertype IPv4 (0x0800), length 370: 10.0.0.2.67 > 10.0.0.57.68: BOOTP/DHCP, Reply, length 328
```

Here we can indeed see the BOOTP/DHCP request send by the VM.

---
**REMEMBER**

That the `tap` interface to which the virtual machine is plugged is the point
where everything that is going from the VM should be visible, ALWAYS. If
something is not visible with tcpdump here, it means that the guest operating
system did not sent it at all and the issue is somewhere in the guest VM for
sure.

---

Now, if we see that this DHCP request is being send from the VM, as next step we
can check if it is visible in the DHCP server's side.
As I mentioned already, Neutron DHCP agent creates a `qdhcp-` namespace and
spawns a dnsmasq server inside it. It also creates `tap` interface which is
internal Openvswitch port plugged directly into the integration bridge
(`br-int`). From that point on, it is wired into the Neutron's network by the
neutron-ovs-agent in exactly same way as a port which belongs to the Virtual
Machine. And if You have access to the host, You can go into the `qdhcp-`
namespace and check there with tcpdump if the DHCP requests are getting to the
dnsmasq service:

```bash
$ sudo ip netns exec qdhcp-5d90176f-d6e4-461e-94c1-c0f44e7cc52b ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host
       valid_lft forever preferred_lft forever
20: tapdcb8a7b0-69: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450 qdisc noqueue state UNKNOWN group default qlen 1000
    link/ether fa:16:3e:65:83:63 brd ff:ff:ff:ff:ff:ff
    inet 10.0.0.2/26 brd 10.0.0.63 scope global tapdcb8a7b0-69
       valid_lft forever preferred_lft forever
    inet6 fdba:b3c4:c275:0:f816:3eff:fe65:8363/64 scope global
       valid_lft forever preferred_lft forever
    inet6 fe80::f816:3eff:fe65:8363/64 scope link
       valid_lft forever preferred_lft forever

sudo ip netns exec qdhcp-5d90176f-d6e4-461e-94c1-c0f44e7cc52b tcpdump -i tapdcb8a7b0-69 -nle not port 22
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on tapdcb8a7b0-69, link-type EN10MB (Ethernet), capture size 262144 bytes
16:02:53.976241 fa:16:3e:3d:0f:f6 > ff:ff:ff:ff:ff:ff, ethertype IPv4 (0x0800), length 342: 0.0.0.0.68 > 255.255.255.255.67: BOOTP/DHCP, Request from fa:16:3e:3d:0f:f6, length 300
16:02:53.976700 fa:16:3e:65:83:63 > fa:16:3e:3d:0f:f6, ethertype IPv4 (0x0800), length 370: 10.0.0.2.67 > 10.0.0.57.68: BOOTP/DHCP, Reply, length 328

```

Here, we can see that the DHCP request is getting to the instance and the DHCP
response is sent back.
In my case everything worked fine and my vm received an IP address from the DHCP
server. But what if the DHCP request wasn't visible in the `tap`
interface in the `dhcp-` namespace? Or dnsmasq wasn't repling?
There are a few more steps that we can take to narrow down where the issue is.

* packets may be lost somewhere in the integration (`br-int`) or tunnel
  (`br-tun`) bridges in one of the nodes or
* there is some problem with the dnsmasq process.

#### Packet flow in the Openvswitch bridges

The first thing to do when we see packets, like DHCP requests, in the `tap` of
the Virtual Machine but not in the `tap` of the `dhcp-` namespace, is to check
if said packets are actually going out from the compute node. We are using a
tunnel network with [vxlan](https://datatracker.ietf.org/doc/html/rfc7348)
encapsulation in this example, so packets through various Open Flow rules in
the `br-int` bridge should be sent to the tunnel bridge (`br-tun`) and be
encapsulated there into UDP packets and sent in turn to the wire. Let's see
what this looks like in detail:

* first, lets check the `ofport` number of the VM interface, remembering that in
  the case when the `iptables_hybrid` firewall driver is used, we are talking
  about a `qvo` interface, not a `tap` (```3e399bf2-31``` is the beginning of the
  port's UUID in Neutron database and it is used in the interface's name):

```bash
sudo ovs-ofctl show br-int | grep 3e399bf2-31
 7(qvo3e399bf2-31): addr:fa:6e:de:ff:f3:66
```

In this case the `ofport` number for this port is `7`.

* Now Open Flow rules in the `br-int` bridge

```bash {lineos=table,hl_lines=[3,4,"10-13",17]}
sudo ovs-ofctl dump-flows br-int
 cookie=0x369da291a727c114, duration=177905.040s, table=0, n_packets=0, n_bytes=0, priority=65535,vlan_tci=0x0fff/0x1fff actions=drop
 cookie=0x369da291a727c114, duration=177282.221s, table=0, n_packets=1, n_bytes=78, priority=10,icmp6,in_port="qvo3e399bf2-31",icmp_type=136 actions=resubmit(,24)
 cookie=0x369da291a727c114, duration=177282.214s, table=0, n_packets=22, n_bytes=924, priority=10,arp,in_port="qvo3e399bf2-31" actions=resubmit(,24)
 cookie=0x369da291a727c114, duration=177904.948s, table=0, n_packets=2579, n_bytes=409087, priority=2,in_port="int-br-ex" actions=drop
 cookie=0x369da291a727c114, duration=177282.238s, table=0, n_packets=2328, n_bytes=218677, priority=9,in_port="qvo3e399bf2-31" actions=resubmit(,25)
 cookie=0x369da291a727c114, duration=177902.256s, table=0, n_packets=23023, n_bytes=3633570, priority=3,in_port="int-br-ex",vlan_tci=0x0000/0x1fff actions=mod_vlan_vid:1,resubmit(,60)
 cookie=0x369da291a727c114, duration=177905.045s, table=0, n_packets=5395, n_bytes=639713, priority=0 actions=resubmit(,60)
 cookie=0x369da291a727c114, duration=177905.047s, table=23, n_packets=0, n_bytes=0, priority=0 actions=drop
 cookie=0x369da291a727c114, duration=177282.231s, table=24, n_packets=0, n_bytes=0, priority=2,icmp6,in_port="qvo3e399bf2-31",icmp_type=136,nd_target=fdba:b3c4:c275:0:f816:3eff:fe3d:ff6 actions=resubmit(,60)
 cookie=0x369da291a727c114, duration=177282.228s, table=24, n_packets=1, n_bytes=78, priority=2,icmp6,in_port="qvo3e399bf2-31",icmp_type=136,nd_target=fe80::f816:3eff:fe3d:ff6 actions=resubmit(,60)
 cookie=0x369da291a727c114, duration=177282.217s, table=24, n_packets=22, n_bytes=924, priority=2,arp,in_port="qvo3e399bf2-31",arp_spa=10.0.0.57 actions=resubmit(,25)
 cookie=0x369da291a727c114, duration=177905.041s, table=24, n_packets=0, n_bytes=0, priority=0 actions=drop
 cookie=0x369da291a727c114, duration=177282.248s, table=25, n_packets=2292, n_bytes=215521, priority=2,in_port="qvo3e399bf2-31",dl_src=fa:16:3e:3d:0f:f6 actions=resubmit(,30)
 cookie=0x369da291a727c114, duration=177905.037s, table=30, n_packets=2415, n_bytes=223875, priority=0 actions=resubmit(,60)
 cookie=0x369da291a727c114, duration=177905.035s, table=31, n_packets=0, n_bytes=0, priority=0 actions=resubmit(,60)
 cookie=0x369da291a727c114, duration=177905.043s, table=60, n_packets=30834, n_bytes=4497236, priority=3 actions=NORMAL
 cookie=0x369da291a727c114, duration=177905.039s, table=62, n_packets=0, n_bytes=0, priority=3 actions=NORMAL
```

I marked with blue the rules related to the traffic
going out from the `vm1`. You can use a tool like `watch` and pay attention
to the `n_packets` field to see the rules packets are going through. Maybe, for some
reason, it hits some rule with `actions:drop` and is dropped? Or it doesn't match
any of the rules in one of the tables so is not processed further and is
effectively dropped too?
Finally, in this case it should hit the rule with `actions:NORMAL` from
`table=60`. That `actions:NORMAL` means that Openvswitch should process
packet as any other switch would do. More about it is in the [Openvswitch man
pages](https://man7.org/linux/man-pages/man7/ovs-actions.7.html). In our case,
packets should be sent through the patch port patch-tun to the tunnel
bridge.

Openflow rules in the br-tun:
```bash {lineos=table,hl_lines=[2,7]}
sudo ovs-ofctl dump-flows br-tun
 cookie=0x3afc26c166dbc0d5, duration=178529.412s, table=0, n_packets=28078, n_bytes=4234292, priority=1,in_port="patch-int" actions=resubmit(,2)
 cookie=0x3afc26c166dbc0d5, duration=178527.151s, table=0, n_packets=313, n_bytes=39021, priority=1,in_port="vxlan-0a780064" actions=resubmit(,4)
 cookie=0x3afc26c166dbc0d5, duration=178520.885s, table=0, n_packets=2006, n_bytes=170922, priority=1,in_port="vxlan-0a780066" actions=resubmit(,4)
 cookie=0x3afc26c166dbc0d5, duration=178529.410s, table=0, n_packets=0, n_bytes=0, priority=0 actions=drop
 cookie=0x3afc26c166dbc0d5, duration=178529.409s, table=2, n_packets=224, n_bytes=31064, priority=0,dl_dst=00:00:00:00:00:00/01:00:00:00:00:00 actions=resubmit(,20)
 cookie=0x3afc26c166dbc0d5, duration=178529.408s, table=2, n_packets=27854, n_bytes=4203228, priority=0,dl_dst=01:00:00:00:00:00/01:00:00:00:00:00 actions=resubmit(,22)
 cookie=0x3afc26c166dbc0d5, duration=178529.407s, table=3, n_packets=0, n_bytes=0, priority=0 actions=drop
 cookie=0x3afc26c166dbc0d5, duration=178498.982s, table=4, n_packets=2319, n_bytes=209943, priority=1,tun_id=0x6ab actions=mod_vlan_vid:2,resubmit(,10)
 cookie=0x3afc26c166dbc0d5, duration=178529.407s, table=4, n_packets=0, n_bytes=0, priority=0 actions=drop
 cookie=0x3afc26c166dbc0d5, duration=178529.406s, table=6, n_packets=0, n_bytes=0, priority=0 actions=drop
 cookie=0x3afc26c166dbc0d5, duration=178529.405s, table=10, n_packets=2319, n_bytes=209943, priority=1 actions=learn(table=20,hard_timeout=300,priority=1,cookie=0x3afc26c166dbc0d5,NXM_OF_VLAN_TCI[0..11],NXM_OF_ETH_DST[]=NXM_OF_ETH_SRC[],load:0->NXM_OF_VLAN_TCI[],load:NXM_NX_TUN_ID[]->NXM_NX_TUN_ID[],output:OXM_OF_IN_PORT[]),output:"patch-int"
 cookie=0x3afc26c166dbc0d5, duration=178486.044s, table=20, n_packets=0, n_bytes=0, hard_timeout=300, priority=1,vlan_tci=0x0002/0x0fff,dl_dst=fa:16:3e:4a:97:e2 actions=load:0->NXM_OF_VLAN_TCI[],load:0x6ab->NXM_NX_TUN_ID[],output:"vxlan-0a780066"
 cookie=0x3afc26c166dbc0d5, duration=178529.401s, table=20, n_packets=4, n_bytes=1368, priority=0 actions=resubmit(,22)
 cookie=0x3afc26c166dbc0d5, duration=178498.992s, table=22, n_packets=4735, n_bytes=559806, priority=1,dl_vlan=2 actions=strip_vlan,load:0x6ab->NXM_NX_TUN_ID[],output:"vxlan-0a780064",output:"vxlan-0a780066"
 cookie=0x3afc26c166dbc0d5, duration=178529.400s, table=22, n_packets=23123, n_bytes=3644790, priority=0 actions=drop
```

There are interesting things happening here. First packets coming from `br-int`
are hitting the rule in `table=0`, which resubmits them to `table 2`, where
there are basically 2 rules: one for broadcast and one for unicast packets. In
the case of the DHCP request, which is broadcast, it matches the first rule and
the packet is resubmitted to `table 22`. In that table the packets that match
the correct vlan_id (2 in our case) have that vlan id stripped off, get the VNI
for the corresponding neutron network attached to them (segmentation_id in the
Neutron network's parameters) and are sent to the vxlan- interfaces that are
connected with other nodes in the cluster.

Now, the reader might ask what is that vlan_id found in the packets.
Basically, neutron uses internally vlan IDs to separate packets from different
networks in each compute node. So that vlan_id=2 in this case is only an internal
thing on that compute node. In another node, packets from the same network may use
a completely different vlan id and that's normal.
To check what vlan_id is used for the network in some host, You can check
the `tag` attribute of the `tap` or the `qvo` port in the integration bridge:

```bash
$ sudo ovs-vsctl list port qvo3e399bf2-31 | grep tag
other_config        : {net_uuid="5d90176f-d6e4-461e-94c1-c0f44e7cc52b", network_type=vxlan, physical_network=None, segmentation_id="1707", tag="2"}
tag                 : 2
```

Another question that might be asked is: how to check if packets were actually
sent through the vxlan tunnel and which interface was used for that?

In regards to the interface, it is the kernel who decides that according to its
routing table, so You can check what IP addresses are used to establish tunnel
connection:

```bash
$ sudo ovs-vsctl show
a66585ea-370a-4a1b-95e2-5083b08c9991
    Manager "ptcp:6640:127.0.0.1"
        is_connected: true
    ...
    Bridge br-tun
        Controller "tcp:127.0.0.1:6633"
            is_connected: true
        fail_mode: secure
        datapath_type: system
        Port gre-0a780066
            Interface gre-0a780066
                type: gre
                options: {df_default="true", egress_pkt_mark="0", in_key=flow, local_ip="10.120.0.101", out_key=flow, remote_ip="10.120.0.102"}
        Port br-tun
            Interface br-tun
                type: internal
        Port vxlan-0a780066
            Interface vxlan-0a780066
                type: vxlan
                options: {df_default="true", egress_pkt_mark="0", in_key=flow, local_ip="10.120.0.101", out_key=flow, remote_ip="10.120.0.102"}
        Port vxlan-0a780064
            Interface vxlan-0a780064
                type: vxlan
                options: {df_default="true", egress_pkt_mark="0", in_key=flow, local_ip="10.120.0.101", out_key=flow, remote_ip="10.120.0.100"}
        Port patch-int
            Interface patch-int
                type: patch
                options: {peer=patch-tun}
    ...
```

and then check which interface is used to reach such IP addresses:

```bash
$ ip route get 10.120.0.100
10.120.0.100 dev br-infra src 10.120.0.101 uid 1000
    cache
```

Now, that we know which interface is used to send the vxlan packets, we can use
tcpdump to check if there are actually sent out:

```bash
$ sudo tcpdump -i br-infra -nel
17:01:58.677926 8e:91:da:21:b2:49 > 56:1e:c3:34:49:40, ethertype IPv4 (0x0800), length 392: 10.120.0.101.48360 > 10.120.0.102.4789: VXLAN, flags [I] (0x08), vni 1707
fa:16:3e:3d:0f:f6 > ff:ff:ff:ff:ff:ff, ethertype IPv4 (0x0800), length 342: 0.0.0.0.68 > 255.255.255.255.67: BOOTP/DHCP, Request from fa:16:3e:3d:0f:f6, length 300
17:01:58.678526 c2:be:74:bf:4d:41 > 8e:91:da:21:b2:49, ethertype IPv4 (0x0800), length 420: 10.120.0.100.59227 > 10.120.0.101.4789: VXLAN, flags [I] (0x08), vni 1707
fa:16:3e:65:83:63 > fa:16:3e:3d:0f:f6, ethertype IPv4 (0x0800), length 370: 10.0.0.2.67 > 10.0.0.57.68: BOOTP/DHCP, Reply, length 328
```

If packets are not sent to the wire here, it means that the issue is somewhere
in the OpenFlow rules either in the `br-int` or `br-tun` bridge.  If packets are
sent out as expected, You can do the same verification in the node running the
DHCP service for the network, to see if they are actially arriving there.  If
they are not - the issue is somwhere outside OpenStack, in the underlay network.

If vxlan packets are arriving to the physical interface in the node with DHCP
server, then we need to investigate Open Flow rules there to see how incomming
traffic is treated.

* first packets are going to the `br-tun`

```bash {lineos=table,hl_lines=[3,9,12]}
sudo ovs-ofctl dump-flows br-tun
 cookie=0x14226289a54c3126, duration=184660.289s, table=0, n_packets=374, n_bytes=46507, priority=1,in_port="patch-int" actions=resubmit(,2)
 cookie=0x14226289a54c3126, duration=180616.030s, table=0, n_packets=5031, n_bytes=600518, priority=1,in_port="vxlan-0a780065" actions=resubmit(,4)
 cookie=0x14226289a54c3126, duration=180609.746s, table=0, n_packets=2053, n_bytes=174534, priority=1,in_port="vxlan-0a780066" actions=resubmit(,4)
 cookie=0x14226289a54c3126, duration=184660.287s, table=0, n_packets=0, n_bytes=0, priority=0 actions=drop
 cookie=0x14226289a54c3126, duration=184660.286s, table=2, n_packets=361, n_bytes=45349, priority=0,dl_dst=00:00:00:00:00:00/01:00:00:00:00:00 actions=resubmit(,20)
 cookie=0x14226289a54c3126, duration=184660.285s, table=2, n_packets=13, n_bytes=1158, priority=0,dl_dst=01:00:00:00:00:00/01:00:00:00:00:00 actions=resubmit(,22)
 cookie=0x14226289a54c3126, duration=184660.284s, table=3, n_packets=0, n_bytes=0, priority=0 actions=drop
 cookie=0x14226289a54c3126, duration=184650.882s, table=4, n_packets=7084, n_bytes=775052, priority=1,tun_id=0x6ab actions=mod_vlan_vid:1,resubmit(,10)
 cookie=0x14226289a54c3126, duration=184660.284s, table=4, n_packets=0, n_bytes=0, priority=0 actions=drop
 cookie=0x14226289a54c3126, duration=184660.283s, table=6, n_packets=0, n_bytes=0, priority=0 actions=drop
 cookie=0x14226289a54c3126, duration=184660.281s, table=10, n_packets=7084, n_bytes=775052, priority=1 actions=learn(table=20,hard_timeout=300,priority=1,cookie=0x14226289a54c3126,NXM_OF_VLAN_TCI[0..11],NXM_OF_ETH_DST[]=NXM_OF_ETH_SRC[],load:0->NXM_OF_VLAN_TCI[],load:NXM_NX_TUN_ID[]->NXM_NX_TUN_ID[],output:OXM_OF_IN_PORT[]),output:"patch-int"
 cookie=0x14226289a54c3126, duration=180581.019s, table=20, n_packets=339, n_bytes=42097, hard_timeout=300, priority=1,vlan_tci=0x0001/0x0fff,dl_dst=fa:16:3e:3d:0f:f6 actions=load:0->NXM_OF_VLAN_TCI[],load:0x6ab->NXM_NX_TUN_ID[],output:"vxlan-0a780065"
 cookie=0x14226289a54c3126, duration=180574.915s, table=20, n_packets=22, n_bytes=3252, hard_timeout=300, priority=1,vlan_tci=0x0001/0x0fff,dl_dst=fa:16:3e:4a:97:e2 actions=load:0->NXM_OF_VLAN_TCI[],load:0x6ab->NXM_NX_TUN_ID[],output:"vxlan-0a780066"
 cookie=0x14226289a54c3126, duration=180258.134s, table=20, n_packets=0, n_bytes=0, hard_timeout=300, priority=1,vlan_tci=0x0001/0x0fff,dl_dst=fa:16:3e:fc:a9:f5 actions=load:0->NXM_OF_VLAN_TCI[],load:0x6ab->NXM_NX_TUN_ID[],output:"vxlan-0a780065"
 cookie=0x14226289a54c3126, duration=184660.280s, table=20, n_packets=0, n_bytes=0, priority=0 actions=resubmit(,22)
 cookie=0x14226289a54c3126, duration=180609.743s, table=22, n_packets=0, n_bytes=0, priority=1,dl_vlan=1 actions=strip_vlan,load:0x6ab->NXM_NX_TUN_ID[],output:"vxlan-0a780065",output:"vxlan-0a780066"
 cookie=0x14226289a54c3126, duration=184660.279s, table=22, n_packets=13, n_bytes=1158, priority=0 actions=drop
```

Here packets are first coming through the rule in `table=0` and are resubmitted
to table=4. In table 4 there is a rule that matches on the `vni` for the network
and based on that sets the corresponding vlan_id in the packet (remember, that
this local vlan can be different on each node for the same network, as explained
above). Next packets are resubmitted to `table=10`, where "magic" happens. There
is rule which learns mac addresses of the source of the packet and based on that
it installs in `table=20` some rules. After that, the packet is send to the
`br-int`.

That additional rule installed in `table=20` helpes later if some packets are
sent in the opposite direction, because it will be known where the destination MAC
actually is, so packets will not be flooded to all the tunnel ends but will be
sent directly to the correct one.

* Last step - integration bridge

```bash {lineos=table,hl_lines=[4,9]}
sudo ovs-ofctl dump-flows br-int
 cookie=0xb7c139d9895c895b, duration=185061.813s, table=0, n_packets=0, n_bytes=0, priority=65535,vlan_tci=0x0fff/0x1fff actions=drop
 cookie=0xb7c139d9895c895b, duration=185060.193s, table=0, n_packets=22775, n_bytes=3593295, priority=2,in_port="int-br-ex" actions=drop
 cookie=0xb7c139d9895c895b, duration=185061.828s, table=0, n_packets=7470, n_bytes=822927, priority=0 actions=resubmit(,60)
 cookie=0xb7c139d9895c895b, duration=185061.829s, table=23, n_packets=0, n_bytes=0, priority=0 actions=drop
 cookie=0xb7c139d9895c895b, duration=185061.822s, table=24, n_packets=0, n_bytes=0, priority=0 actions=drop
 cookie=0xb7c139d9895c895b, duration=185061.809s, table=30, n_packets=0, n_bytes=0, priority=0 actions=resubmit(,60)
 cookie=0xb7c139d9895c895b, duration=185061.808s, table=31, n_packets=0, n_bytes=0, priority=0 actions=resubmit(,60)
 cookie=0xb7c139d9895c895b, duration=185061.823s, table=60, n_packets=7470, n_bytes=822927, priority=3 actions=NORMAL
 cookie=0xb7c139d9895c895b, duration=185061.812s, table=62, n_packets=0, n_bytes=0, priority=3 actions=NORMAL
```

Here it is very easy - the packet is going through the rule in `table=0` that
resubmits it to `table=60` where it hits the flow with `actions=NORMAL`.
Everything else is then done internally by openvswitch.

One more thing - do You remember about that local vlan_id which was added to the
packet in the br-int? The `tap` interface in the `br-int` also has set `tag` to
that value and this means that only packets with that vlan_id will be sent to
that port by openvswitch and vlan tag will be automatically stripped off before
the packet goes outside openvswitch. That's why there is no vlan tag visible in
the packets inside Virtual Machines, or inside the `qdhcp-` namespace.

#### Dnsmasq logs

If the DHCP requests are arriving correctly to the `tap` interface in the
`qdhcp-` namespace, but there is no DHCP reply, it may mean that the problem
is somewhere in dnsmasq. To check that You can look in the dnsmasq logs in the
journal log. If everything is working properly, there should be something like the following in the
log:

```bash
journalctl -f | grep dhcp
Jan 21 21:06:28 devstack-ubuntu-controller dnsmasq-dhcp[322219]: DHCPDISCOVER(tapdcb8a7b0-69) fa:16:3e:3d:0f:f6
Jan 21 21:06:28 devstack-ubuntu-controller dnsmasq-dhcp[322219]: DHCPOFFER(tapdcb8a7b0-69) 10.0.0.57 fa:16:3e:3d:0f:f6
Jan 21 21:06:28 devstack-ubuntu-controller dnsmasq-dhcp[322219]: DHCPREQUEST(tapdcb8a7b0-69) 10.0.0.57 fa:16:3e:3d:0f:f6

```

That is basically whole packet flow between a Virtual Machine and the DHCP
service when tunnel networks are used in Neutron. If all of that works fine,
Your instance should have a properly configured IP address and should be
accessible in the network.

### Configuration of the Virtual Machine - debugging metadata connectivity issues

Now, once the Virtual Machine gets its IP address from the DHCP server, usually
next step during the boot process is to connect to the [metadata
service](https://docs.openstack.org/nova/latest/user/metadata.html) so that
[cloud-init](https://cloudinit.readthedocs.io/en/latest/) can configure things,
like e.g. SSH keys.

Cloud-init looks for the metadata service at `169.254.169.254` by sending HTTP
requests to that address. In OpenStack the way metadata is provided to the
Virtual Machines is as follows:

![Access to metadata service](/images/debugging-neutron/legacy-routing/metadata_access.png)

Metadata for servers are stored in Nova and that's why there are so many
steps in access to them. Lets go through that process step by step
now:

* First the Virtual Machine send a regular HTTP request to the
  `http://169.254.169.254` address:

```bash
$ curl http://169.254.169.254
1.0
2007-01-19
2007-03-01
2007-08-29
2007-10-10
2007-12-15
2008-02-01
2008-09-01
2009-04-04
```

* When network is connected to a Neutron router, the metadata service address is
  simply reachable through default gateway configured in the Virtual Machine, so
  requests are basically sent to the Neutron router's namespace, called
  `qrouter-<router_id>` and created by the L3 agent on the controller or
  dedicated networker node.
  Lets now focus a bit on how things are configured in the router's namespace.
  For each subnet plugged to the router, there is an interface called `qr-XXX`
  created there and it is very similar to the `tap` port created in the
  `qdhcp-` namespace. The only difference between them is the naming
  convention, other than that it's the same internal port in the integration
  bridge (`br-int`).
  To find out in which node a router is hosted, the Neutron API can be used:

```bash
$ openstack network agent list --router router1
+--------------------------------------+------------+----------------------------+-------------------+-------+-------+------------------+
| ID                                   | Agent Type | Host                       | Availability Zone | Alive | State | Binary           |
+--------------------------------------+------------+----------------------------+-------------------+-------+-------+------------------+
| a1520b60-8f2d-4602-8c9d-0f1bf7febc19 | L3 agent   | devstack-ubuntu-controller | nova              | :-)   | UP    | neutron-l3-agent |
+--------------------------------------+------------+----------------------------+-------------------+-------+-------+------------------+
```

In this case it is `devstack-ubuntu-controller` node. So lets go to that node
and check what's in the `qrouter-` namespace:

```bash
$ sudo ip netns exec qrouter-50711ca1-1e58-44dc-bd95-a87250d5b4ae ip a
...
14: qr-2ff5e542-74: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450 qdisc noqueue state UNKNOWN group default qlen 1000
    link/ether fa:16:3e:fc:a9:f5 brd ff:ff:ff:ff:ff:ff
    inet6 fdba:b3c4:c275::1/64 scope global
       valid_lft forever preferred_lft forever
    inet6 fe80::f816:3eff:fefc:a9f5/64 scope link
       valid_lft forever preferred_lft forever
15: qr-7297041d-55: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450 qdisc noqueue state UNKNOWN group default qlen 1000
    link/ether fa:16:3e:e3:70:9e brd ff:ff:ff:ff:ff:ff
    inet 10.0.0.1/26 brd 10.0.0.63 scope global qr-7297041d-55
       valid_lft forever preferred_lft forever
    inet6 fe80::f816:3eff:fee3:709e/64 scope link
       valid_lft forever preferred_lft forever
16: qg-abce46ac-61: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UNKNOWN group default qlen 1000
    link/ether fa:16:3e:0b:68:73 brd ff:ff:ff:ff:ff:ff
    inet 10.10.0.240/25 brd 10.10.0.255 scope global qg-abce46ac-61
       valid_lft forever preferred_lft forever
    inet 10.10.0.151/32 brd 10.10.0.151 scope global qg-abce46ac-61
       valid_lft forever preferred_lft forever
    inet6 2001:db8::2c0/64 scope global
       valid_lft forever preferred_lft forever
    inet6 fe80::f816:3eff:fe0b:6873/64 scope link
       valid_lft forever preferred_lft forever
```

There are 2 `qr-` interfaces there because there are 2 subnets plugged into that
router (one IPv4 and one IPv6 subnet). There is also `qg-` interface, which is
the gateway to the external world for the router, but about that we will talk
later.

Now, lets get back to the metadata request. In the `qrouter-` namespace there
is a `haproxy` service running. This is the `Metadata proxy` block in the picture
above:

```bash
ps aux | grep haproxy
vagrant    17147  0.0  0.0  89168  4372 ?        Ssl  11:50   0:00 haproxy -f /opt/stack/data/neutron/ns-metadata-proxy/50711ca1-1e58-44dc-bd95-a87250d5b4ae.conf
```

There are also `iptables` rules configured to send requests from `169.254.169.254`
to the haproxy instance, which listens on port `9697`:

```bash
sudo ip netns exec qrouter-50711ca1-1e58-44dc-bd95-a87250d5b4ae iptables-save
...
*nat
...
-A neutron-l3-agent-PREROUTING -d 169.254.169.254/32 -i qr-+ -p tcp -m tcp --dport 80 -j REDIRECT --to-ports 9697
...
*filter
...
-A neutron-l3-agent-INPUT -m mark --mark 0x1/0xffff -j ACCEPT
-A neutron-l3-agent-INPUT -p tcp -m tcp --dport 9697 -j DROP
-A neutron-l3-agent-scope -o qr-7297041d-55 -m mark ! --mark 0x4000000/0xffff0000 -j DROP
COMMIT
# Completed on Mon Jan 24 12:04:41 2022
```

Of course, there are many more iptables rules configured in the `qrouter-`
namespace. I included here only those related to the metadata service.
So `iptables` redirects HTTP requests to port `9697` so they can get to the
haproxy instance where an additional header `X-Neutron-router-ID` is added to them. This
additional header is added so Neutron can easily find the port from
which the requests were made by matching on the sender's IP address and the router-id
(or network-id if it's all done on the `qdhcp-` namespace, but lets not focus
on that case for now).

---
**REMEMBER**

If for any reason the HTTP requests made from the instance are sent with an IP
address different to the address allocated to the port in the Neutron database,
then Neutron will not be able to find the id of the port and thus an `HTTP 404`
error will be replied.

---

* When the `neutron-metadata-agent` receives the modified request from
  `haproxy`, it sends an RPC request to the neutron server to get data about the
  port attached to the Virtual Machine. In the port's information there is
  always an attribute called `device_id` and this is the uuid of the virtual
  machine in the Nova database.

* Finally the `neutron-metadata-agent` sends a HTTP request to the
  `nova-metadata-api` to get the metadata for that specific virtual machine.

* After the `neutron-metadata-agent` receives response from the
  `nova-metadata-api` it sends it back to the haproxy and from there it goes to
  the virtual machine.

#### Isolated networks

The above example describes the typical use case when a Neutron's project network is
plugged into a Neutron router. But there may be also the case when the network is
isolated and not plugged to any router. In such case metadata can be provided by
the `neutron-dhcp-agent`. In this situation the `haproxy` service is spawned in the
`qdhcp-` namespace and bound to the `tap` port there (the same one on which
`dnsmasq` works. The IP address `169.254.169.254` is in this case configured
directly on the `tap` interface too, so there is no need to use iptables to
redirect requests to the haproxy instance.

I didn't describe really how packets with such HTTP requests flow through the
bridges and nodes to reach the `qrouter` namespace, because it's exactly the
same as in the case of DHCP requests.


### One Virtual Machine talks to the another one - debugging issues with connectivity between 2 VMs in the same L2 network

Now, when virtual machines are spawned and configured by cloud-init
properly, they should also be able to communicate to each other.
Packet flow between 2 instances is exactly the same as between virtual machine
and e.g. DHCP namespace and the `tap` interface there. The only difference is
that instead of `tap` interface in the `qdhcp` namespace on one side, there
are virtual machines plugged through the `qbr` bridge to the integration
bridge.
When there are problems with type of communication, You need to carry out the
same investigation with tools like tcpdump and ovs-ofctl as described above in
the *[debugging DHCP](#Creating-new-Virtual-Machine---debugging-DHCP)* section.

### Connect to the Virtual Machine from the Internet

The last thing I want to cover in this article is connectivity of
Virtual Machines to the external world.
As it is shown in the picture above, VM1 and VM2 are connected directly only to
the private, tunnel network. This network doesn't have access to external world
directly.

But this private network is connected to the router which has configured also
something what is known in Neutron as the `external gateway`:

```bash
$ openstack router show router1
+-------------------------+-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| Field                   | Value                                                                                                                                                                                                                                                                             |
+-------------------------+-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
...
| external_gateway_info   | {"network_id": "b12c783e-9439-4967-890a-c87762a3f04b", "external_fixed_ips": [{"subnet_id": "eeb7856c-73ef-405f-8dc5-e836b46bc73a", "ip_address": "10.10.0.240"}, {"subnet_id": "c9d5090f-d99e-40f5-a249-b2f8c313d984", "ip_address": "2001:db8::2c0"}], "enable_snat": true}     |
...
| id                      | 50711ca1-1e58-44dc-bd95-a87250d5b4ae                                                                                                                                                                                                                                              |
...
| name                    | router1                                                                                                                                                                                                                                                                           |
+-------------------------+-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
```

That external network can provide access to the external world for the instances
plugged into the private networks.
There are 3 possible ways to provide such access:

* by attaching a public IP address, called a `Floating IP` to the port which
  belongs to the Virtual Machine - that way the VM has access to the external
  world using that Floating IP but also it can be reachable (e.g. SSH to it is
  possible) from the external world,

* by using the `SNAT` functionality of the router - in this way the VM can reach
  external world using the external_gateway's IP address (`10.10.0.240` in the
  example above) but it can't be accessible from the external network in any
  way,

* using `port_forwarding` functionality which enables the sharing of one
  Floating IP address among many VMs, by routing incomming traffic directed to
  specific TCP/UDP ports to TCP/UDP ports in those VMs.

In this article I will focus on the first 2 cases just mentioned. I will not
describe port forwarding.

#### Debugging connectivity through the Floating IPs

Lets see what happens when a Virtual Machine wants to reach out to some external
service, for example ping the `8.8.8.8` IP address.
First the IP address should be reachable through the default gateway configured
in the instance:

```bash
ip route
default via 10.0.0.1 dev eth0
10.0.0.0/26 dev eth0 scope link  src 10.0.0.57
```

So, when we ping `8.8.8.8` from the instance:

```bash
$ ping 8.8.8.8 -c 1
PING 8.8.8.8 (8.8.8.8): 56 data bytes
64 bytes from 8.8.8.8: seq=0 ttl=111 time=46.180 ms

--- 8.8.8.8 ping statistics ---
1 packets transmitted, 1 packets received, 0% packet loss
round-trip min/avg/max = 46.180/46.180/46.180 ms
```

the packets go through the private network to the `qrouter` namespace flowing
through bridges and tunnels in the exact same way as when the VM communicates
with the DHCP server, the metadata server or another VM connected to the same
tunnel network.

```bash
$ sudo ip netns exec qrouter-50711ca1-1e58-44dc-bd95-a87250d5b4ae tcpdump -i qr-7297041d-55 -nvl
tcpdump: listening on qr-7297041d-55, link-type EN10MB (Ethernet), capture size 262144 bytes
13:11:48.800330 IP (tos 0x0, ttl 64, id 4862, offset 0, flags [DF], proto ICMP (1), length 84)
    10.0.0.57 > 8.8.8.8: ICMP echo request, id 49665, seq 0, length 64
13:11:48.841181 IP (tos 0x80, ttl 111, id 0, offset 0, flags [none], proto ICMP (1), length 84)
    8.8.8.8 > 10.0.0.57: ICMP echo reply, id 49665, seq 0, length 64
```

In the `qrouter` namespace there are iptables rules to do NAT:

```bash
$ sudo ip netns exec qrouter-50711ca1-1e58-44dc-bd95-a87250d5b4ae iptables-save
# Generated by iptables-save v1.8.4 on Mon Jan 24 13:13:49 2022
*nat
...
-A neutron-l3-agent-OUTPUT -d 10.10.0.151/32 -j DNAT --to-destination 10.0.0.57
-A neutron-l3-agent-PREROUTING -d 10.10.0.151/32 -j DNAT --to-destination 10.0.0.57
```

On the external gateway interface it should be visible as:

```bash
$ sudo ip netns exec qrouter-50711ca1-1e58-44dc-bd95-a87250d5b4ae tcpdump -i qg-abce46ac-61 -nvl
tcpdump: listening on qg-abce46ac-61, link-type EN10MB (Ethernet), capture size 262144 bytes
13:16:37.822219 IP (tos 0x0, ttl 63, id 16879, offset 0, flags [DF], proto ICMP (1), length 84)
    10.10.0.151 > 8.8.8.8: ICMP echo request, id 1391, seq 0, length 64
13:16:37.864548 IP (tos 0x80, ttl 112, id 0, offset 0, flags [none], proto ICMP (1), length 84)
    8.8.8.8 > 10.10.0.151: ICMP echo reply, id 1391, seq 0, length 64
13:16:39.100308 IP (tos 0x0, ttl 64, id 65454, offset 0, flags [DF], proto UDP (17), length 72)
    10.120.0.1.57621 > 10.120.0.255.57621: UDP, length 44

```

As it can be seen here, thanks to the address translation that took place in the
router, the `ICMP` echo request is sent with `10.10.0.151` (Floating IP in
Neutron) as the source IP address.

#### Connection with the external network

Here there is something completely new coming on. The packets are going through
the `qg-` interface to the Neutron network called `public` which, unlike
`private`, is not a tunnel network. `public` is marked in Neutron as an
`external` network, which means that it can be used as the external_gateway for
routers. It is a provider network and can be of one of two possible types:

* flat - network without any segmentation id used,
* vlan - uses vlans.

`provider` networks have that designation because they need to be configured by
the cloud provider in the underlay physical networking fabric. Packets sent from
the OpenStack nodes through such networks go through the underlay physical
fabric directly.  In the case of `vlan` networks, packets go to the physical
datacenter network with the configured vlan_id, so the datacenter network
devices need to know about that vlan_id.
In this example, a `flat` network is used, but differences between them are really
minor and I will describe them later.

##### Packets flow in the provider networks

Like `tap-` and `qr-` interfaces mentioned earlier, interface `qg-` from the
router namespace is a port of type `internal` in Openvswitch. It is also plugged
into the integration bridge `br-int`.

Lets examine Open Flow rules for traffic originating from such `qg-` interface:

```bash {lineos=table,hl_lines=[5,10]}
$ sudo ovs-ofctl dump-flows br-int
 cookie=0xc609e5deded3773a, duration=7102.716s, table=0, n_packets=0, n_bytes=0, priority=65535,vlan_tci=0x0fff/0x1fff actions=drop
 cookie=0xc609e5deded3773a, duration=5754.608s, table=0, n_packets=767, n_bytes=116954, priority=3,in_port="int-br-ex",vlan_tci=0x0000/0x1fff actions=mod_vlan_vid:2,resubmit(,60)
 cookie=0xc609e5deded3773a, duration=7096.975s, table=0, n_packets=700, n_bytes=50245, priority=2,in_port="int-br-ex" actions=drop
 cookie=0xc609e5deded3773a, duration=7102.740s, table=0, n_packets=1823, n_bytes=180601, priority=0 actions=resubmit(,60)
 cookie=0xc609e5deded3773a, duration=7102.749s, table=23, n_packets=0, n_bytes=0, priority=0 actions=drop
 cookie=0xc609e5deded3773a, duration=7102.726s, table=24, n_packets=0, n_bytes=0, priority=0 actions=drop
 cookie=0xc609e5deded3773a, duration=7102.711s, table=30, n_packets=0, n_bytes=0, priority=0 actions=resubmit(,60)
 cookie=0xc609e5deded3773a, duration=7102.706s, table=31, n_packets=0, n_bytes=0, priority=0 actions=resubmit(,60)
 cookie=0xc609e5deded3773a, duration=7102.735s, table=60, n_packets=2590, n_bytes=297555, priority=3 actions=NORMAL
 cookie=0xc609e5deded3773a, duration=7102.714s, table=62, n_packets=0, n_bytes=0, priority=3 actions=NORMAL
```

Basically this traffic is just sent to `table=60` where the `NORMAL` action is
performed. This means that packets will go through the `int-br-ex` patch port,
which connects the integration bridge with the the proper external bridge, which
in this case is named `br-ex`.

Open Flow rules in the `br-ex` bridge looks like:

```bash {lineos=table,hl_lines=[2]}
sudo ovs-ofctl dump-flows br-ex
 cookie=0xb255f8dd88f646a2, duration=5971.389s, table=0, n_packets=257, n_bytes=20074, priority=4,in_port="phy-br-ex",dl_vlan=2 actions=strip_vlan,NORMAL
 cookie=0xb255f8dd88f646a2, duration=7313.746s, table=0, n_packets=319, n_bytes=32060, priority=2,in_port="phy-br-ex" actions=drop
 cookie=0xb255f8dd88f646a2, duration=7313.770s, table=0, n_packets=1496, n_bytes=171835, priority=0 actions=NORMAL
```

There is only one important rule for us here. It's the first one. It matches
packets which come from the `phy-br-ex` port (the other end of the patch port
`int-br-ex`) with `vlan_id=2` (this is the local vlan_id used by
neutron-ovs-agent). For such packets it strips vlan id and performs `NORMAL`
action. That action basically means in this case to send packets to the physical
interface which should be one of the ports in that bridge.

---
**REMEMBER**

Each external bridge, like `br-ex` needs to be configured manually by the cloud
operator and needs to have a physical interface inside. Otherwise packets will
not be sent to the wire and there will be no connectivity with the external
world using such network.

---

Here we can see the small difference between `flat` and `vlan` networks. In
the case of `vlan` networks, instead of the `strip_vlan` action in the OpenFlow
rule, there is `mod_vlan_vid:<segmentation_id_vlan>`.

#### Debugging connectivity from the external world to the VM using Floating IPs

When there is a `Floating IP` associated with the Virtual Machine, it can be
reached from outside as well. In this case, traffic is basically going from
the physical interface to the `br-ex` bridge where it is simply processed by
the `NORMAL` action:

```bash
 cookie=0xb255f8dd88f646a2, duration=7313.770s, table=0, n_packets=1496, n_bytes=171835, priority=0 actions=NORMAL
```

and sent to the integration bridge.

In the integration bridge it goes through the rules:

```bash {lineos=table,hl_lines=[3,10]}
$ sudo ovs-ofctl dump-flows br-int
 cookie=0xc609e5deded3773a, duration=7102.716s, table=0, n_packets=0, n_bytes=0, priority=65535,vlan_tci=0x0fff/0x1fff actions=drop
 cookie=0xc609e5deded3773a, duration=5754.608s, table=0, n_packets=767, n_bytes=116954, priority=3,in_port="int-br-ex",vlan_tci=0x0000/0x1fff actions=mod_vlan_vid:2,resubmit(,60)
 cookie=0xc609e5deded3773a, duration=7096.975s, table=0, n_packets=700, n_bytes=50245, priority=2,in_port="int-br-ex" actions=drop
 cookie=0xc609e5deded3773a, duration=7102.740s, table=0, n_packets=1823, n_bytes=180601, priority=0 actions=resubmit(,60)
 cookie=0xc609e5deded3773a, duration=7102.749s, table=23, n_packets=0, n_bytes=0, priority=0 actions=drop
 cookie=0xc609e5deded3773a, duration=7102.726s, table=24, n_packets=0, n_bytes=0, priority=0 actions=drop
 cookie=0xc609e5deded3773a, duration=7102.711s, table=30, n_packets=0, n_bytes=0, priority=0 actions=resubmit(,60)
 cookie=0xc609e5deded3773a, duration=7102.706s, table=31, n_packets=0, n_bytes=0, priority=0 actions=resubmit(,60)
 cookie=0xc609e5deded3773a, duration=7102.735s, table=60, n_packets=2590, n_bytes=297555, priority=3 actions=NORMAL
 cookie=0xc609e5deded3773a, duration=7102.714s, table=62, n_packets=0, n_bytes=0, priority=3 actions=NORMAL
```

The most important and interesting rule here is:

```
 cookie=0xc609e5deded3773a, duration=5754.608s, table=0, n_packets=767, n_bytes=116954, priority=3,in_port="int-br-ex",vlan_tci=0x0000/0x1fff actions=mod_vlan_vid:2,resubmit(,60)
```

It matches packets comming from the `int-br-ex` patch port and have no vlan_id
(`vlan_tci=0x0000/0x1fff`). For those packets it sets vlan_id=2 (again, this is
the `vlan_id` used locally by the neutron-ovs-agent and can be different on each
node for the same network, or can even be changed after restarting openvswitch).
Then it is send to the `table 60` where it hits the rule with action `NORMAL`
again, so it is sent to the proper interface, which in our case will be the
`qg-` port.


#### Debugging connectivity issues through the SNAT

For connectivity using SNAT most of the parts are exactly the same as when
Floating IP is used. The only differences here are in the iptables rules in the
`qrouter` namespace:

```bash
$ sudo ip netns exec qrouter-50711ca1-1e58-44dc-bd95-a87250d5b4ae iptables-save
...
# Generated by iptables-save v1.8.4 on Mon Jan 24 14:28:27 2022
*nat
...
-A neutron-l3-agent-snat -j neutron-l3-agent-float-snat
-A neutron-l3-agent-snat -o qg-abce46ac-61 -j SNAT --to-source 10.10.0.240 --random-fully
-A neutron-l3-agent-snat -m mark ! --mark 0x2/0xffff -m conntrack --ctstate DNAT -j SNAT --to-source 10.10.0.240 --random-fully
-A neutron-postrouting-bottom -m comment --comment "Perform source NAT on outgoing traffic." -j neutron-l3-agent-snat
COMMIT
...
# Completed on Mon Jan 24 14:28:27 2022
# Generated by iptables-save v1.8.4 on Mon Jan 24 14:28:27 2022
*mangle
...
-A neutron-l3-agent-mark -i qg-abce46ac-61 -j MARK --set-xmark 0x2/0xffff
...
```

Also, traffic is sent through the `qg-` interface with a different source IP
address. As there is no IP address assigned only to that specific Virtual
Machine, the router's external_gateway's address is used:

```bash
sudo ip netns exec qrouter-50711ca1-1e58-44dc-bd95-a87250d5b4ae tcpdump -i qg-abce46ac-61 -nvl
tcpdump: listening on qg-abce46ac-61, link-type EN10MB (Ethernet), capture size 262144 bytes
14:32:37.398487 IP (tos 0x0, ttl 63, id 41329, offset 0, flags [DF], proto ICMP (1), length 84)
    10.10.0.240 > 8.8.8.8: ICMP echo request, id 6160, seq 0, length 64
14:32:37.442202 IP (tos 0x80, ttl 112, id 0, offset 0, flags [none], proto ICMP (1), length 84)
    8.8.8.8 > 10.10.0.240: ICMP echo reply, id 6160, seq 0, length 64
```


#### Bridge mappings

I mentioned above that physical bridges, like `br-ex`, need to have a physical
interface plugged to be able to send traffic to the corresponding physical
network.  There can be more than one physical bridge in the node, if the node
has access to more than one physical network.

Now, how does Neutron know which physical bridge should be used for the Neutron
network? To tell that to neutron-ovs-agent, there is a very important
configuration setting in the neutron-ovs-agent. It is called `bridge_mappings`.
This config option is used to configure which bridge has access to what physical
network (Neutron doesn't know which NICs are plugged to such physical network,
that's why NIC information needs to be added to the bridge by the operator of the
cloud).
Networks have an attribute in the Neutron database called
`provider:physical_network` which has to match with one of the physical
networks set in the bridge_mappings configuration option - that way Neutron knows
which bridge should be used to connect to which provider network.

## Next steps

Here I wanted to write also about High Availiability routers (L3-ha) and how
security groups are implemented by the `neutron-ovs-agents` using different
drivers but this post is already very long so I will write another posts for
those additional things.

I hope this is somewhow useful for You and that it will help understand how
Neutron with the ML2 and OVS backend does networking in general. If You have any
suggestions or questions about it, please contact me via IRC or email - see
[contact](https://kaplonski.pl/about/) for contact details.

## Special Thanks

Special thanks to [Miguel Lavalle](https://www.miguellavalle.com/?) who helped
me a lot by reviewing this article!
