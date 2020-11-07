---
title: "Virtual PTG October 2020 - summary of the Neutron sessions"
date: 2020-11-02T23:16:59+01:00
tags: ["openstack"]
draft: false
---

Yet another virtual PTG happened between 26th and 30th of October. Neutron team
had sessions in each day of the PTG. Below is summary of what we discussed and
agreed during that time.  Etherpad with notes from the discussions can be found
at [OpenDev Etherpad](https://etherpad.opendev.org/p/neutron-wallaby-ptg).

## Retrospective of the Victoria cycle
From the good things during _Victoria_ cycle team pointed:
* Complete 8 blueprints including the
  [Metadata over IPv6](https://blueprints.launchpad.net/neutron/+spec/metadata-over-ipv6),
* Improved feature parity in the OVN driver,
* Good review velocity

From the not so good things we mentioned:
* CI instability - average number of rechecks needed to merge patch in last year
  can be found at [ImgBB](https://ibb.co/12sB9N9),
* Too much "Red Hat" in the Neutron team - and percentage of the reviews (and
  patches) done by people from Red Hat is constantly increasing over last few
  cycles. As a main reason of that we pointed that it is hard to change
  companies way of thinking that dedicating developer to the upstream project
  means that they lose developer in downstream.
* Migration of our CI and Zuul v3 took us a lot of time and effort - but good
  thing is that we accomplished that in the Victoria cycle :)

During that session we also agreed on some action items for the Wallaby cycle:
* keep talking about OVN details - Miguel Lavalle and me will work on some way
  to deliver talks about Neutron OVN backend and OVN internals to the community
  during next months. The idea is to maybe propose some talks ever 6 weeks or
  so. This may make ovn driver development more diverse and let operators to
  thing about migration to the ovn backend.

## Review of the existing meetings
We reviewed list of our existing upstream meetings and discussed about ideas on
how to increase number of attendees on the meetings.
We decided to:
* drop neutron-qos meeting as it's not needed anymore
* advertise more meetings and meetings' agenda on the OpenStack mailing list - I
  will send reminders with links to the agenda before every meeting
* Together with Lajos Katona we will give some introduction to the debugging of
  CI issues in Neutron*

## Support for old versions
Bernard started discussion about support for the old releases in Neutron and
Neutron stadium projects.
For Neutron we decided to mark __Ocata__ branch as unmaintained already as its
gate is already broken.
For the __Pike__ and never branches we will keep them in the __EM__ phase as
there is still some community interest to keep those branches open.
For the stadium projects we decided to do it similary to what we did while
looking for new maintainers for the projects. We will send email "call for
maintainers" for such stable branches. If there will be no voluneers to step in,
fix gate issues and keep those branches healthy, we will mark them as
__unmaintained__ and later as __End of Life__ (EOL).
Currently broken CI is in projects:
* networking-sfc,
* networking-bgpvpn/bagpipe,
* neutron-fwaas

And those are candidates to be marked as unmaintained if there will be no
volunteers to fix them.
Bernard Cafarelli volunteered to work on that in next days/weeks.

## Healtcheck API endpoint
We discussed as our healtcheck API should works. During the discussion we
decided that:
* healtcheck result should __NOT__ rely on the agents status, it should rely on
  worker's ability to connect to the DB and MQ (rabbitmq)
* Lajos will ask community (API experts) about some guidance how it should works
  on the whole OpenStack level,
* As for reference implementation we can check e.g. [Octavia](
  https://opendev.org/openstack/octavia/src/branch/master/octavia/api/healthcheck/healthcheck_plugins.py)
  and [Keystone](https://docs.openstack.org/keystone/victoria/admin/health-check-middleware.html)
  which already implemented it.

## Squash of the DB migration script
Rodolfo explained us what are benefits of doing such squash of the db migration
scripts from the old versions:
* Deployment is faster: we don't need to create/delete tables or create+update
  other ones - the win is small possibly in the magnitude of 5s per job,
* DB definition is centralized in one place, not in original definition plus
  further migrations - that is most important reason why we really should do
  that,
* UTs faster: removal of some older checks.

The problem with this may be that we need to do that carefully and be really
verbose about with such changes we may break stadium projects or 3rd party
projects which are doing db migration too.
To minimalize potential breakage, we will announce such changes on the OpenStack
discuss mailing list.
Rodolfo volunteered to take propose squash up to Liberty release in W cycle.
Together with this squash we will also document that process so in next cycles
we should be able to do squashes for newer releases in easier way.
Lajos volunteered to help with fixing Neutron stadium projects if that will be
needed.

## Switch to the new engine facade
We were discussing how to move on and finally finish old [Blueprint](
https://blueprints.launchpad.net/neutron/+spec/enginefacade-switch). We
decided that together with Rodolfo we will try how this new engine facade will
work without using transaction guards in the code. Hopefully that will let us
move on with this. If not, we will try to reach out to some DB experts for some
help with this.

## Change from rootwrap to the privsep
This is now community goal during the Wallaby cycle so we need to focus on it to
accomplish that transition finally.
This transition may speed up and make our code a bit more secure.
Rodolfo explained us multiple possible strategies of migration:
* move to native, e.g.
  * replace ps with python psutils, not using rootwrap or privsep
  * replace ip commands with pyroute2, under a privsep context (elevated
    permissions needed)
* directly translate rootwrap to privsep, executing the same shell command but
  under a privsep context

To move on with this I will create list of the pieces of code which needs to be
transitioned in the Neutron repo and in the stadium projects.
Current todo items can be found on
[the StoryBoard](https://storyboard.openstack.org/#!/story/2007686).

## Migration to the NFtables
During this session we were discussing potential strategies on how to migrate
from the old iptables to the new nftables. We need to start planning that work
as it major Linux distributions (e.g. RHEL) are planning to deprecate iptables
in next releases.
It seems that currently all major distros (Ubuntu, Centos, OpenSuSE) supports
nftables already.
We decided that in Wallaby cycle we will propose new _Manager_ class and we will
add some config option which will allow people to test new solution.
In next cycles we will continue work on it to make it stable and to make upgrade
and migration path for users as easy as possible.
There is already created [blueprint](https://blueprints.launchpad.net/neutron/+spec/nftables-migration)
to track progress on that topic.
We need to migrate:
* Linuxbridge firewall, iptables OVS hybrid firewall,
* L3 code (legacy router, DVR router, conntrack, port forwarding),
* iptables metering,
* metadata proxy,
* dhcp agent for when it does metadata for isolated networks and namespace
  creation,
* neutron-vpnaas - ipsec code,
* and maybe something else what we didn't found yet.

## Nova-Neutron cross project session
We had very interesting discussion with Nova team. We were discussing topics
like:
* NUMA affinity in the neutron port
* vhost-vdpa support
* default __vnic_type__/__port flavour__

Notes from that discussion are available in the nova's
[etherpad](https://etherpad.opendev.org/p/nova-wallaby-ptg).

## Neutron scalling issues
At this session we were discussing issues mentioned by operators on the Forum
sessions a week before the PTG. There was couple of issues mentioned there:
* problems with retries of the DB operations - we should migrate all our code to
  the oslo.db retries mechanism - new
  [blueprint](https://blueprints.launchpad.net/neutron/+spec/oslo-db-retrie)
  is created to track

  progress on that one.
* problems with maintenance of the agents, like e.g. DHCP or L3 agents - many of
  those issues are caused by how our agents are designed and to really fix that
  we would need very deep and huge changes. But also many of those issues can be
  solved by the __ovn__ backend - **and that is strategic direction in which
  neutron wants to go in the next cycles**,
* Miguel Lavalle and I volunteered to do some profiling of the agents to see
  where we are loosing most of the time - maybe we will be able to find some
  _low hanging fruits_ which can be fixed and improve the situation at least a
  bit,
* Similar problem with neutron-ovs-agent and especially security groups which
  are using _remove group id_ as a reference - here we also need some volunteers
  who will try to optimize that.

## CI (in)stablility
On Thursday we were discussing how to improve our very poor CI. Finally we
decided to:
* not recheck patches without giving reason of recheck in the comment - there
  should be already reported bug which should be linked in the _recheck_
  comment, or user should open new one and link to it also. IN case if the
  problem was e.g. related to infra some simple comment like _infra issue_ will
  also be enough there,
* To lower number of existing jobs we will do some changes like:
  * move *-neutron-lib-master and *-ovs-master jobs to the experimental and
    periodic queues to not run them on every patch,
  * I will switch _neutron-tempest-plugin-api_ job to be deployed with uwsgi so
    we can drop _neutron-tempest-with-uwsgi_ job,
  * Consolidate _neutron-tempest-plugin-scenario-linuxbridge_ and
    _neutron-tempest-linuxbridge_ jobs,
  * Consolidate _neutron-tempest-plugin-scenario-iptables_hybrid and
    _neutron-tempest-iptables_hybrid jobs,

Later we also discussed about the way how to run or skip tests which can be only
run when some specific feature is available in the cloud (e.g. _Metadata over
IPv6_). After some discussion we decided to add new config option with list of
enabled features. It will be very similar to the existing option
_api_extensions_. Lajos volunteered to work on that.

As last CI related topic we discussed about testing DVR in our CI. Oleg Bondarev
volunteered to check and try to fix broken
_neutron-tempest-plugin-dvr-multinode-scenario_ job.

## Flow based DHCP

Liu Yulong raised topic about new way of doing fully distributed DHCP service,
instead of using _DHCP agent_ on the nodes -
proposed [RFE](https://bugs.launchpad.net/neutron/+bug/1900934) is proposed.
His proposal of doing OpenFlow based DHCP (similar to what e.g. ovn-controller
is doing) is described in
[slides](https://github.com/gotostack/shanghai_ptg/blob/master/shanghai_neutron_ptg_topics_liuyulong.pdf).
It could be implemented as an L2 agent extension and enabled by operators in the
config when they would need it.  As a next step Liu will now propose spec with
details about this solution and we will continue discussion about it in the
spec's review.

## Routed provider networks limited to one host

As a lost topic on Thursday we briefly talked about old
[RFE](https://bugs.launchpad.net/neutron/+bug/1764738).
Miguel Lavalle told us that his company, Verizon Media, is interested in working
on this RFE in next cycles. This also involves some work on Nova's side which
was started by Sylvain Bauza already. Miguel will sync with Sylvain on that RFE.

## L3 feature improvements

On Friday we were discussing some potential improvements in the L3 area. Lajos
and Bence shown us some features which their company is interested in and on
which they plan to work. Those are things like:
* support for Bidirectional Forwarding Detection
* some additional API to set additional router parameters like:
  * ECMP max path,
  * ECMP hash algorith
* --provider-allocation-pool parameter in the subnets - in some specific cases
  it may help to use IPs from such _special_ pool for some infrastructure needs,
  more details about that will come in the RFE in future,
For now all those described above improvements are in very early planning phase
but Bence will sync with Liu and Liu will dedicate some time to discuss progress
on them during the __L3 subteam meetings__.

## Leveraging routing-on-the-host in Neutron in our next-gen clusters

As a last topic on Friday we were discussing potential solutions of the _L3 on
the host_ in the Neutron. The idea here is very similar to what e.g. __Calico
plugin__ is doing currently.
More details about potential solutions are described in the
[etherpad](https://etherpad.opendev.org/p/neutron-routing-on-the-host).
During the discussion Dawid Deja from OVH told us that OVH is also using very
similar, downstream only solution.
Conclusion of that discussion was that we may have most of the needed code
already in Neutron and some stadium projects so as a first step people who are
interested in that topic, like Jan Gutter, Miguel and Dawid will work on some
deployment guide for such use case.

## Team photo
During the PTG we also made team photos.

![team_photo_1](/images/Virtual_PTG_October_2020/neutron_ptg_team_photo_october_2020.png)

![team_photo_2](/images/Virtual_PTG_October_2020/neutron_ptg_team_photo_october_2020_with_masks.png)

![team_photo_3](/images/Virtual_PTG_October_2020/screenshot_1.png)

![team_photo_4](/images/Virtual_PTG_October_2020/screenshot_2.png)

![team_photo_5](/images/Virtual_PTG_October_2020/screenshot_3.png)

![team_photo_6](/images/Virtual_PTG_October_2020/screenshot_4.png)

## Summary
In overall it was another great and very productive PTG. Of course I missed a
lot all in person meetings, talks during the lunch or coffee breaks and most
Neutron team dinner but currently it was all what we can do. I hope that next
year we will be able to meet in person finally.
Take care of Yourself and see You online :)
