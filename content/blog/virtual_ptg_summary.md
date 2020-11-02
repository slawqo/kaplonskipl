---
title: "My summary of the OpenStack Virtual_PTG July 2020"
date: 2020-06-10T22:59:35+01:00
draft: false
---

## Retrospective
From the good things team mentioned that migration of the networking-ovn driver
to the core neutron went well. Also our CI stability improves in the last cycle.
Another good thing was that we implemented all required in this cycle community
goals and we even migrated almost all jobs to Zuul v3 syntax already. Not so
good was progress on some important Blueprints, like adoptoion of the new engine
facade. The other thing mentioned here was activity in the stadium projects and
in the neutron-lib.

## Support for the new oslo.policy
Akihiro and Miguel volunteered to work on this in the Victoria if they will have
some cycles. It is important feature from the Keystone team’s point of view.
Keystone and Nova teams already have some experience on that transition so they
can help us with that. We need to ensure that new policies in neutron can works
fine with e.g. old ones defined in the stadium projects.

## Adoption of new enginefacade
We discussed how to finish this adoption of new enginefacade in Neutron in
Victoria cycle. There is couple of people who volunteered to help with that.
After it will be done in Neutron we need to check what changes are required in
the stadium projects too.

## tenant_id/project_id transition
Rodolfo raised this issue that in some places we may still have e.g. only
tenant_id and miss project_id. We should all be aware of that and whenever we
see something like that e.g. in neutron, neutron-lib, tempest or other repos, we
should propose patch to change that.

## Neutronclient -> OSC migration
We found out that the only blocker which prevents us to EOL python-neutronclient
as a CLI tool is lack of possibility to pass custom arguments to the commands.
As we know, OSC/SDK team don’t have anything against adding support for that in
OSC. It’s actually already done for e.g. Glance. Slaweq volunteered to work on
it this cycle and if that will be done, define final date of EOL for
python-neutronclient for Victoria+2 cycle.

## RFEs review
We reviewed about 10 old, stagnant RFEs. We decicded to close some of them as
non relevant anymore. For others we discussed some details and we will continue
this discussion on LP and on the drivers meetings.

## Review of stadium projects
We needed to review list of stadium projects again. Conclusions are:

* neutron-vpnaas, networking-odl, networking-bagpipe, networking-bgpvpn are in
  good shape and we can definitely keep them in stadium,
* neutron-fwaas - we will delete all code from this repo now and mark it as
  deleted project, see Congress for example how this is done,
* neutron-dynamic-routing - send email to call for maintainers, but don’t consider
  it as deprecated yet in this cycle, maybe in the next one,
* networking-midonet - this one is the problem currently as it isn’t well tested
  with current neutron in u/s gate.

## Meeting with Edge SIG
During this call Neutron-interconnection project was mentioned. For now it was
moved to unofficial projects namespace but maybe someone will revive this
project if there is use case for it. We talked about RFE and doing it in
Victoria cycle. After discussions about the same with Nova team on Friday (see
below) we decided that as a first step Nova will work on changes on their side
and those changes will not require anything from the Neutron. On our side,
Miguel Lavalle volunteered tentatively that he can work on this on too. There
was also question about support of IEEE 1588 (PTP) in Neutron - so far we don’t
have any support for it. But we are always welcome for new RFEs :)

## Future of metering agent
Discussion was related to the spec. General conclustion from the discussion is
that we will work on this new L3 agent extension which will collect metering
data but this will not be in any way replacement for the old metering agent.
When this will be ready, we can start discussion about potential deprecation of
the old metering agent and its API but that’s nothing for the current cycle for
sure.

## ML2/OVS and OVN feature gaps
List of gaps between ML2/OVS and ML2/OVN backends is in the doc. We discussed
how to maintain this list and make it smaller. Decision from this discussion is
that we will try to remember to add new items to this list every time when some
new feature will be added to Neutron and implemented in ML2/OVS. We will also
open LP for such things to keep it tracked on OVN driver side and to implement
that when it will be needed by community.

## OVN driver and agent API
During this discussion we talked about Neutron agents API and two options in it:

* setting admin to be disabled (admin_state_up = False) - we decided that this is
  really relevant only for L3 and DHCP agents. It’s not really used for OVS or
  Linuxbridge agents and it shouldn’t also have any effect for OVN drivers. So
  Kuba will propose RFE with description of what our API should return in case of
  setting this admin_state_up value for agents for which it doesn’t makes sense,
* deletion of agents - we decided that for ovn agents it should be possible to
  delete them in same way how it is done now for other agents. This has some
  downside when user will delete agent which is still alive. In such case agent
  will be deleted and immediatelly shown back in neutron agent list. That is
  because of how OVN is storing data about those “agents” internally and how
  Neutron OVN driver uses this data. This isn’t however big issue because it may
  happend in exactly same way for example for OVS/DHCP/L3 agents if user will
  delete them just before it will send new heartbeat message to the Neutron
  server.

## Stateless security groups support for OVS firewall (and OVN?)
Conclusion from this discussion is that there is need and use cases (DPDK,
telco) to introduce stateless security groups in openvswitch and ovn firewall
drivers. During the discussion we agreed that implementation shouldn’t be really
thought. Bernard volunteered to open LP to track it for both of those drivers.

## OVN concept of distributed localport ports used for Metadata agent and DHCP
Maciek explained us limitations in current implementation of segments in Neutron
and how it blocks OVN driver to support that. Basically the problem is that
currently Neutron has limitation that one port can be only on one segment. OVN
driver is using “special” ports (local ports in OVN) to handle metadata and
dhcp. And such port has to have IP address from all segments which belongs to
the network. We agreed to introduce new type of device_owner which will mean
that port is “distributed”. Such kind of port may then be in more than one
segment at same time. Basically this concept is similar to how DVR router ports
works currently.

## Switching default backend in Devstack to be OVN
At the end of the day we discussed possibility to switch Devstack’s default
backend in Neutron to be OVN instead of OVS agent. We all agreed that this is
big step forward and that we want to do that as Neutron team. We want to give
with this change clear signal that we believe that OVN backend is future of the
Neutron and we want to invest more in that. Hovewer we want to highlight that
this means nothing for real production environments. We are not going to
deprecate any backends. All existing drivers will be tested as they are tested
now in the Neutron gate. Different deployment tools may still use different
backends as default ones. This change will be however pretty big change for all
OpenStack projects which will use OVN driver as Neutron’s backend in their jobs.
Such change was already done in TripleO in Stein cycle and it went pretty smooth
so we are not expecting any serious issues there. We would like to do this
change which we set for ourselves is Victoria-2 milestone.

## Deprecation of use_veth_interconnection in the ovs-agent
It seems after discussion that this config option was introduced very long time
ago and was probably related to something like descibed in the bug but later
Open vSwitch introduced patch ports and there was another old LP bug which was
saying that ovs-agent should use patch ports instead of veth pairs. And that
solution is still recommended today in the Neutron. So we agreed to deprecate
this option in the Victoria cycle and remove it completly in W cycle.

## Loki service plugin
Service plugin called “loki” is developed to be use in the CI jobs and to add DB
deadlock errors to some random DB queries. But for now it’s not used at all in
the Neutron CI. During the discussion we decided to add new periodic Zuul job
which will run neutron-tempest-plugin.api tests with enabled loki service plugin
to check if Neutron still can handle properly such random DB Deadlocks.

## SR-IOV ML2 driver support for VLAN trunking
Patch related to this discussion is at https://review.opendev.org/#/c/665467/72/

* it’s big patch which introduces base trunk driver interface for SR-IOV agent but
  also implementation of this base driver for Intel cards. During the discussion
  we agreed that we shouldn’t have such vendor specific code in the Neutron tree.
  Our suggestion was to split this patch into two pieces - base driver which will
  be in the Neutron repo and Intel specific driver which will be hosted in some
  other repository.

## Neutron-lib
We were discussion couple of different issues related to the neutron-lib in
general.

* Nate started discussion about future of the neutron-lib and if this is still
  really needed. His proposal was to relax a bit policies about what should be
  re-homed from Neutron to the neutron-lib repository. Originally neutron-lib was
  introduced to help neutron stadium and 3rd-party projects to not import things
  from neutron but from the neutron-lib only and have less dependencies thanks to
  that. Currently development of neutron-lib is much slower than it was in the
  past and we don’t see so much need to use neutron-lib from the developers of the
  stadium or 3rd-party projects. After some discussion we agreed that neutron-lib
  is still usefull and important to keep e.g. api definitions and highligh base
  generic interfaces for neutron and other related projects. So we will keep
  neutron-lib as it is now but we will relax a bit policy of what will be moved to
  this repo from the neutron. Second outcome from the same discussion was decision
  that we will change a bit our policy of updating every neutron-lib/neutron
  consumer when we will re-home things. We will be focused only on Neutron stadium
  projects. For other projects we will introduce 1 cycle deprecation period for
  things which are moved out from the neutron to neutron-lib repo.

* Adam from the Octavia team started discussion about how to test neutron and
  neutron-lib changes. Currently all neutron CI jobs are using latest released
  neutron-lib version. But because of that developers can’t use “Depends-On” to
  test some neutron change together with the related neutron-lib change. After
  discussion our final decision is that we will propose one additional CI job
  which will use neutron-lib from master branch. So this job can be used to check
  neutron and neutron-lib changes together before neutron-lib with required change
  will be released.

## Nova-Neutron-Cyborg cross projects session
This was long (2h) session where we discussed couple of important things.

* First topic was about scheduling of the routed networks. After some discussion
  was that Nova team will work on something only on Nova’s side as a first step.
  Later we will see how to move forward with integration of Nova and Neutron
  there.
* Second topic was about updating minimum bandwdith for the ports. We agreed that
  this should be done on Neutron side and Neutron will update resource allocation
  directly in Placement. There is no need to involve Nova in this process. We also
  decided that we will not allow update values of the existing, already associated
  rules in the QoS Policy. The only possible way to change minimum bandwidth will
  be to update port to associate it with new QoS Policy.
* Last topic on this meeting was related moslty to Cyborg and Nova cooperation.
  There will be required some changes on the Neutron side as well, like e.g. add
  new api extension which will add “device profile” to the port object. This new
  attribute will contain e.g. information about name of the device profile from
  Cyborg. Details are in the doc

## Future of lib/neutron and lib/neutron-legacy in devstack
In Devstack there are currently 2 modules which can configure Neutron. Old one
called “lib/neutron-legacy” and the new one called “lib/neutron”. It is like
that since many cycles that “lib/neutron-legacy” is deprecated. But it is still
used everywhwere. New module isn’t still finished and isn’t working fine. This
is very confusing for users as really maintained and recommended is still
“lib/neutron-legacy” module.

During the discussion Sean Collins explained us that originally this new module
was created as an attempt to refactor old module, and to make Neutron in the
Devstack better to maintain. But now we see that this process failed as new
module isn’t still used and we don’t have any cycles to work on it. So our final
conclusion is to “undeprecate” old “lib/neutron-legacy” and get rid of the new
module.

## OVN master and stable CI jobs
Currently we are running all Neutron OVN related jobs with OVN installed from
packages provided by operating system. Maciek explained us why we should also
test it with OVN from master branch. It is like that because often new features
added to the OVN driver in Neutron are really dependent on changes in the OVN
project. And until such changes in OVN aren’t really released we can’t test if
Neutron OVN changes really works with new OVN code as they should. So we deviced
to change “neutron-ovn-tempest-full-multinode-ovs-master” that it will also
install OVN from the source.

## CI Status
During discussion about our CI jobs we decided to remove from the Neutron check
queue 2 non-voting and still not stable jobs:
“neutron-tempest-plugin-dvr-multinode-scenario” and
“neutron-tempest-dvr-ha-multinode-full” and move those jobs to the experimental
queue. This is temporary solution until we will fix those 2 jobs finally.

## Team photo
It wasn’t as great as in other PTGs. But here are our team photos in the current
context:

![team_photo_1](/images/Virtual_PTG_2020/photo_1.png)

![team_photo_2](/images/Virtual_PTG_2020/photo_2.png)

![team_photo_3](/images/Virtual_PTG_2020/photo_3.png)

Credits for photos comes to Rodolfo and Jakub who made them :) Thx a lot!

## Etherpads
Etherpad from Neutron sessions can be found at
https://etherpad.opendev.org/p/neutron-victoria-ptg. Notes from the
Nova-Neutron-Cyborg session can be found at
https://etherpad.opendev.org/p/nova-victoria-ptg
