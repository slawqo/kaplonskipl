---
title: "My Summary of OpenStack PTG in Shanghai"
date: 2019-11-15T13:56:24+01:00
draft: false
---

This is my summary of OpenStack PTG which had place in Shanghai in November
2019. It is brief summary of all discussions which we had in Neutron room
during 3 days event.

## On boarding

Slides from onboarding session can be found at
[here](https://www.slideshare.net/SawomirKaposki/neutron-on-boarding-room)
In my opinion onboarding was good. There was around 20 (or even more) people in
the room during this session. Together with Miguel Lavalle we gave talk about

- what is Neutron,
- how it works,
- what are main concepts in Neutron,
- how Neutron community works in upstream,
- and how new people can start contributing to the project.

I hope that this will bring us at least some new contributors in Ussuri cycle :)

After the onboarding session we started "real work" at PTG. Below is desription
of various topic which we were talking about during Wednesday, Thursday and
Friday.

## Train retrospective

Good things in Train cycle:

- working with this team is still good experience
- core team is stable, and we didn't lost any core reviewers during the cycle,
- networking is still one of key reasons why people use OpenStack

### Not good things

- dimished vitality in stadium projects - we had also forum session and follow
  discussion about this later during the PTG,
- gate instability - we have seen many issues which were out of our control,
  like infra problems, grenade jobs failures, other projects failures, but also
  many bugs on our side,
- we have really a lot of jobs in our check/gate queue. If each of them is
  failing 5% of times, it's hard to merge any patch as almost every time, one of
  jobs will fail. Later during the PTG we also discussed that topic and we were
  looking for some jobs which we maybe can potentially drop from our queues. See
  below for summary about that,

### Action items/improvements:

- too many team meetings each week. We decided to limit number of meetings by:
  - consolidate performance subteam meeting into weekly team meeting - this topic
    will be added to the team meeting's agenda for team meetings on Monday,
  - consolidate ovn convergence meeting into weekly team meeting - this topic
    will be added to the team meeting's agenda for team meetings on Tuesday,
  - we need to check if QoS subteam meeting is still needed,

- Review process: list of actual review priorities would be useful for the team,
  we will add "Review-Priority" label to the Neutron reviews board and try to
  use it during the Ussuri cycle.

## Openvswitch agent enhancements

We had bunch of topics related to potential improvements for
neutron-openvswitch-agent proposed mostly by Liu Yulong. Slides with his
proposals are available
[here](https://github.com/gotostack/shanghai_ptg/blob/master/shanghai_neutron_ptg_topics_liuyulong.pdf)

### Retire DHCP agent

Resyncs of DHCP agent are problematic, especially when
agent hosts many networks. Proposal was to add new L2 agent's extension which
could be used instead of "regular" DHCP agent and to provide only basic DHCP
functionalities.
Such solutions would work in the way quite similar to how networking-ovn works
today but we would need to implement and maintain own dhcp server application.

Problems of this solution are:

- problems with compatibility e.g. with Ironic,
- how it would work with mixed deployments, e.g. with ovs and sriov agents,
- support for dhcp options,

Advantages of this solution:

- fully distributed DHCP service,
- no DHCP agents, so less RPC messages on the bus and easier maintanance of
  the agents,

Team's feedback for that is that this is potentially nice solution which may
helps in some specific, large scale deploymnets. We can continue discussion
about this during Ussuri cycle for sure.

### Add accepted egress fdb flows

We agreed that this is a bug and we should continue work on this to propose
some way to fix it.
Solution proposed by LIU during this discussion wasn't good as it could
potentially break some corner cases.

### new API and agent for L2 traffic health check

The team asked to add to the spec some more detailed and concrete use cases
with explanation how this new API may help operator of the cloud to
investigate where the problem actually is.

### Local flows cache and batch updating

The team agreed that as long as this will be optional solution which operator
can opt-in we can give it a try. But spec and discuss details there will be
necessary.

### Stop processing ports twice in ovs-agent

We all agreed that this is a bug and should be fixed. But we have to be
careful as fixing this bug may cause some other problems e.g. with
live-migration - see nova-neutron cross project session.

### ovs-agent: batch flow updates with --bundle

We all agreed that this can be done as an improvement of existing code.
Similar option is already used in openvswitch firewall driver.

## Neutron - Cyborg cross project session

Etherpad for the session is at
[here](https://etherpad.openstack.org/p/Shanghai-Neutron-Cyborg-xproj).
Cyborg team wants to include Neutron in workflow of spawning VMs with Smart NICs
or accelerator cards. From Neutron's side, required change is to allow including
"accel" data in port binding profile. As long as this will be well documented
what can be placed there, there should be no problem with doing that.
Technically we can place almost anything there.

## Neutron - Kuryr cross project session

Etherpad for the session is
[here](https://etherpad.openstack.org/p/kuryr-neutron-nice-to-have).
Kuryr team proposed 4 improvements for Neutron which would help a lot Kuryr.
Ideas are:

- Network cascade deletion,
- Force subport deletion,
- Tag resources at creation time,
- Security group creation with rules & bulk security group rule creation

All of those ideas makes sense for Neutron team. Tag resources at creation time
is even accepted rfe already - see
[RFE](https://bugs.launchpad.net/neutron/+bug/1815933) but there was no
volunteer to implement it.
We will add it to list of our BPs tracked weekly on team meeting. Miguel
Lavalle is going to take a look at it during this cycle.
For other proposals we need to have RFEs reported first.

## Starting the process of removing ML2/Linuxbridge

Currently in Neutron tree we have 4 drivers:

- Linuxbridge,
- Openvswitch,
- macvtap,
- SR-IOOV.

SR-IOV driver was out of discussion there as this driver is
addressing slightly different use case than other out drivers.

We started discussion about above topic because we don't want to end up with too
many drivers in-tree and we also had some discussions (and we have spec for that
already) about include networking-ovn as in-tree driver.
So with networking-ovn in-tree we would have already 4 drivers which can be used
on any hardware: linuxbridge, ovs, macvtap and ovn.

Conclusions from the discussion are:

- each driver requires proper testing in the gate, so we need to add many new
  jobs to our check/gate queue,
- currently linuxbridge driver don't have a lot of development and feature
  parity gaps between linuxbridge and ovs drivers is getting bigger and bigger
  (e.g. dvr, trunk ports),
- also macvtap driver don't have a lot of activity in last few cycles. Maybe
  this one could be also considered as candidate to deprecation,
- we need to have process of deprecating some drivers and time horizon for such
  actions should be at least 2 cycles.
- we will not remove any driver completly but rather we will move it to be in
  stadium process first so it still can be maintained by people who are
  interested in it.

Actions to do after this discussion:

- Miguel Lavalle will contact RAX and Godaddy (we know that those are
  Linuxbridge users currently) to ask about their feedback about this,
- if there are any other companies using LB driver, Nate Johnston is willing to
  help conctating them, please reach to him in such case.
- we may ratify marking linuxbridge as deprecated in the team meeting during
  Ussuri cycle if nothing surprising pops in.

## Encrypted(IPSec) tenant networks

Interesting topic proposed but we need to have RFE and spec with more detailed
informations about it to continue discussions.

## Medatada service over IPv6

This is continuation of old [RFE](https://bugs.launchpad.net/neutron/+bug/1460177).
The only real problem is to choose proper IPv6 address which will be well known
address used e.g. by cloud-init.
Original spec proposed fe80::a9fe:a9fe as IPv6 address to access metadata
service.
We decided to be bold and define the standard.
Bence Romsics and Miguel Lavalle volunteered to reach out to cloud-init
maintainers to discuss that.

## Walkthrough of OVN

Since some time we have in review spec about ml2/ovs and ovn convergence. See
[patch](https://review.opendev.org/#/c/658414/) for details.
List of parity gaps between those backends is available in the
[etherpad](https://etherpad.openstack.org/p/ML2-OVS-OVN-Convergence).

During the discussion we talked about things like:

- migration from ml2/ovs to ml2/ovn - some scripts are already done in
  [networking-ovn repo](https://github.com/openstack/networking-ovn/tree/master/migration),
- migration from ml2/lb to ml2/ovn - there was no any work done in this topic so
  far but it should be doable also if someone would need it and want to invest
  own time for that,
- include networking-ovn as in-tree neutron driver and reasons why it could be
  good idea.
  Main reasons of that are:

  - that would help growing networking-ovn community,
  - would help to maintain a healthy project team,
  - the default drivers have always been in-tree,

  However such inclusion may also hurt modularity/logical separation/dependency
  management/packaging/etc so we need to consider it really carefully and
  consider all points of view and opinions.

Next action item on this topic is to write more detailed summary of this topic
and send it to ML and ask wider audience for feedback.

## IPv6 devstack tempest test configuration vs OVN

Generally team supports idea which was described during this session and we
should change sligtly IPv6 config on e.g. devstack deployments.

## Neutron - Edge SIG session

We discussed about [RFE](https://bugs.launchpad.net/neutron/+bug/1832526).
This will require also changes on placement side.
Check [this email](http://lists.openstack.org/pipermail/openstack-discuss/2019-October/009991.html)
for details.
Also some cyborg and ovn related changes may be relevant to topics related to
Edge.
Currently specs which we have are only related to ML2/OVS solution.

## Neutron - Nova cross project session

Etherpad for this session is [here](https://etherpad.openstack.org/p/ptg-ussuri-xproj-nova-neutron).
Summary written already by gibi can be found in
[the email](http://lists.openstack.org/pipermail/openstack-discuss/2019-November/010654.html).
[Here](https://imgur.com/a/12PrQ9W) You can find image which shows in visual way
problem with live-migration of instances with SR-IOV ports.

## Policy handling in Neutron

The goal of the session was to plan on Neutron's side similar effort to what
services like nova are doing now to use new roles like reader and scopes, like
project, domain, system provided by Keystone.
Miguel Lavalle volunteered to work on this for Neutron and to be part of popup
team for cross project collaboration on this topic.

## Neutron performance improvements

Miguel Lavalle shown us his [new profiling decorator](https://review.opendev.org/678438)
and how we all can use it to profile some of API calls in Neutron.

## Reevaluate Stadium projects

This was follow up discussion after forum session. Notes from forum session can
be found in [the etherpad](https://etherpad.openstack.org/p/PVG-Neutron-stadium-projects-the-path-forward).
Nate also prepared some good data about stadium projects activity in last
cycles. It can be found at [sheet](https://ethercalc.openstack.org/neutron-stadium-train-metrics)
and [here](https://ibb.co/SBzDGdD) for details.
We all agreed that projects which are in (relatively) good condition now are:

- networking-ovn,
- networking-odl,
- ovsdbapp

Projects in bad condition are other projects, like:

- neutron-interconnection,
- networking-sfc,
- networking-bagpipe/bgpvpn,
- networking-midonet,
- neutron-fwaas and neutron-fwaas-dashboard,
- neutron-dynamic-routing,
- neutron-vpnaas and neutron-vpnaas-dashboard,

We decided to immediately remove neutron-interconnection project as it was never
really implemented.

For other of those projects, we will send emails to ML to ask for potential
maintainers of those projects. If there will be no any volunteers to maintain
some of those projects, we will deprecated them and move to "x/" namespace in 2
cycles.

## Floating IP's On Routed Networks

There is still interest of doing this. Lajos Katona started adding some scenario
tests for routed networks already as we need improved test coverage for this
feature.
Miguel Lavalle said that he will possibly try to work on implementing this in
Ussuri cycle.

## L3 agent enhancement

We talked about couple potential improvements of existing L3 agent, all proposed
by LIU Yulong.

- retire metering-agent
  It seems that there is some interest in metering agent recently so we
  shouldn't probably consider of retiring it for now.
  We also talked about adding new "tc based" driver to the metering agent and
  this discussion can be continue on
  [RFE](https://bugs.launchpad.net/neutron/+bug/1817881)

- Centralized DNAT (non-DVR) traffic (floating IP) Scale-out
  This is proposal of new DVR solution. Some details of this new solution are
  available [here](https://imgur.com/a/6MeNUNb).
  We agreed that this proposal is trying to solve some very specific use case,
  and it seems to be very complicated solution with many potential corner cases
  to address. As a community we don't want to introduce and maintain such
  complicated new L3 design.

- Lazy-load agent side router resources when no related service port
  Team wants to see RFE with detailed description of the exact problem which
  this is trying to solve and than continue discussion on such RFE.

## Zuul jobs

In this session we talked about jobs which we can potentially promote to be
voting (and we didn't found any of such) and about jobs which we maybe can
potentially remove from our queues.
Here is what we agreed:

- we have 2 iptables_hybrid jobs - one on Fedora and one on Ubuntu - we will
  drop one of those jobs and left only one of them,
- drop neutron-grenade job as it is running still on py27 - we have grenade-py3
  which is the same job but run on py36 already,
- as it is begin of the cycle, we will switch in devstack neutron uwsgi to be
  default choice and we will remove "-uwsgi" jobs from queue,
- we should compare our single node and multinode variants of same jobs and
  maybe promote multinode jobs to be voting and then remove single node job - I
  volunteered to do that,
- remove our existing experimental jobs as those jobs are mostly broken and
  nobody is run those jobs in experimental queue actually,
- Yamamoto will check failing networking-midonet job and propose patch to make
  it passing again,
- we will change neutron-tempest-plugin jobs for branch in EM phase to always
  use certain tempest-plugin and tempest tag, than we will remove those jobs
  from check and gate queue in master branch,

## Stateless security groups

Old [RFE](https://bugs.launchpad.net/neutron/+bug/1753466)
was approved for neutron-fwaas project but we all agreed that this
should be now implemented for security groups in core Neutron.
People from Nuage are interested in work on this in upstream.
We should probably also explore how easy/hard it will be to implement it in
networking-ovn backend.

## Old, stagnant specs

During this session we decided to abandon many of old specs which were proposed
long time ago and there is currently no any activity and interest in continue
working on them.
If anyone would be interested in continue work on some of them, feel free to
contact neutron core team on irc or through email and we can always reopen such
patch.

## Community Goal things

We discussed about currently proposed community goals and who can take care of
which goal on Neutron's side.
Currently there are proposals of community goals as below:

- python3 readiness - Nate will take care of this,
- move jobs definitions to zuul v3 - I will take care of it. In core neutron and
  neutron-tempest-plugin we are (mostly) done. On stadium projects' side this
  will require some work to do,
- Project specific PTL and contributor guides - Miguel Lavalle will take care of
  this goal as former PTL,

We will track progress of community goals weekly in our team meetings.

## Neutron-lib

As some time ago our main neutron-lib maintainer (Boden) leaved from the
project, we need some new volunteers to continue work on it. The list of things
todo is available at the
[etherpad](https://etherpad.openstack.org/p/neutron-lib-volunteers-and-punch-list)
This should be mostly important for people who are maintaining stadium projects
or some 3rd party drivers/plugins so if You are doing things like that, please
check list from
[etherpad](https://etherpad.openstack.org/p/neutron-lib-volunteers-and-punch-list)
and reach out to us on ML or #openstack-neutron IRC channel.

## Team dinner

On Tuesday evening we also had team dinner together. It was really great
experince and pleasure to be with all people from Neutron team and with some
friends from other teams, like folks from Octavia or QA teams :)
Below are some pictures from our team dinner.

![team_dinner_photo_1](/images/Shanghai_PTG/IMG_5593.jpg)

![team_dinner_photo_2](/images/Shanghai_PTG/IMG_5596.jpg)

![team_dinner_photo_3](/images/Shanghai_PTG/IMG_4185.jpeg)

## The end

And that's the end of this summary. Thank You if You read it all up to this
point.
At the end I want also to thank all people who were in Neutron room during this
PTG and who participated in discussions. It couldn't be so effective without all
of You :)

And I also want to say thank You to some people who helped me most in Shanghai:

- **LIU Yulong** - for all Your help with organisation of team dinner.
  It couldn't be done without You :)
- **Miguel Lavalle** - for all Your tips&tricks and for guiding me before
  start of the PTG and during whole week in Shanghai.

