---
title: "Analyzing number of failed builds per patch in OpenStack projects"
date: 2019-12-30T23:56:20+01:00
tags: ["openstack"]
draft: false
---

## Short background

For the past few years I have been an OpenStack Neutron contributor, core
reviewer and now even the project's PTL. One of my responsibilities in the
community is taking care of our CI
system for Neutron. As part of this job I have to constantly check how various
CI jobs are working and if the reasons for their failure are related to the
patch or the CI itself.

This is in general a very time consuming task as Neutron a wide variety of jobs
in both check and gate queues in Zuul. As I write this I'm writing this post,
there are 22 voting jobs in check queue and 19 jobs in gate queue.

The reason why we have so many jobs is that Neutron is very pluggable, so there
are many different L2 backends (like ``openvswitch``, ``linuxbridge`` and soon
we
will also have ``ovn`` as in-tree driver which has to be tested as well),
different firewall drivers (e.g. for openvswitch backend there is
``iptables_hybrid`` and ``openvswitch`` driver). Next there are also different
L3 agent topologies (``legacy``, ``HA``, ``DVR``, ``DVR-HA``). Because all of
that we ended up with many scenario jobs running tempest or
neutron-tempest-plugin
tests on different topologies of Neutron.

Part of my regular daily tasks is preparation of [Neutron CI
meetings](http://eavesdrop.openstack.org/#Neutron_CI_team) and I then always
check our [Grafana
dashboard](http://grafana.openstack.org/d/Hj5IHcSmz/neutron-failure-rate?orgId=1)
to check the overall stability of each job in both check and gate queues.
For a long time I have had a feeling that this does not give us a full picture
of the overall state of the Neutron CI. It was like that because even if
all of the voting jobs had failure rates around (or below) 10%, I saw many
``-1`` votes from Zuul and many ``recheck`` comments in various patches. That is
pretty obvious because if we have more than 20 voting jobs, even if each of them
fails around 10% of the time for random reasons, some jobs will fail almost
every run of Zuul.

## Idea of new metric

So I thought that having a metric which will show just how many times a patch
needed to be ``rechecked`` before it was merged would be useful to get the
overall
status of CI.

The number of ``rechecks`` is in fact something that directly impacts
every developer who is doing patches to the project so it is also useful to
check how annoying it is for contributors to get their patches merged.
And that might also be an important reason why we don't have many new
contributors as
such people can be disappointed and discouraged after sending their first patch
and
will not continue contributing to the project later.

## Script to get data from Gerrit

Unfortunately I didn't find such a metric directly in
[Graphite](https://graphite01.opendev.org/). But good news for me was that
Gerrit gives us [query
API](https://gerrit-review.googlesource.com/Documentation/cmd-query.html) which
can be used to get e.g. all patches for a certain time period, and then the
comments on these patches can be parsed to check how many times it got "Build
failed" comment from
"Zuul" user.

Based on that and on [Assaf Muller's
script](https://github.com/assafmuller/gerrit_time_to_merge/blob/master/time_to_merge.py)
I wrote a simple script which is available in [My github
repo](https://github.com/slawqo/tools/blob/master/rechecks.py) This is simple
script which takes from Gerrit data about patches merged in last X days in a
given
project (e.g. "openstack/neutron"). For each patch it then checks
the last revision number and checks how many times on last revision Zuul
commented "Build failed".

This script is checking only last patches because if build failed on this last
revision it clearly means that this wasn't a problem with the patch itself but
with
the CI job. I also didn't want to check how many times someone wrote "recheck"
comment as sometimes people are doing recheck even if the build was fine, for
example to check if their change fixes some intermittent problem in CI.

Also this script calculates average number of failed builds per week.

## Results

### Neutron

As I have the script I ran it on patches from the last 500 days. First I of
course did
it for the Neutron project. To get this data I ran the script with command like:

    python3 rechecks.py openstack/neutron --newer-than 500 > neutron_500d.csv

Script normally prints comma separated values on stdout so I saved it in
file.

Results are on the image below:
![neutron_500d](/images/post_rechecks/neutron_500d.svg)

Those results aren't good. In fact in Neutron we really sucks with our CI as
in the best weeks developers needs to recheck about 1  or 2 times before
patch
is merged.

### Neutron-lib

I was curious how same metric will look like for smaller project with much less
CI jobs in check and gate queues in Zuul. I checked same data for neutron-lib
project. Results are below:

![neutron-lib_500d](/images/post_rechecks/neutron-lib_500d.svg)

Here results are much better. In most weeks, patches were merged at first CI
run.

My next step was to compare this data with other "big" OpenStack projects. I
checked same data for Nova, Glance and than Cinder. Results are on charts below.

### Nova
![nova_500d](/images/post_rechecks/nova_500d.svg)

### Glance
![glance_500d](/images/post_rechecks/glance_500d.svg)

### Cinder
![cinder_500d](/images/post_rechecks/cinder_500d.svg)

## Conclusions

It seems that this isn't only problem of Neutron. Other projects with many CI
jobs in the queues are also failing quite often. But indeed Neutron seems to be
one of the "bad leaders" in that metric.
One of the reasons why it is like that are probably slow nodes on which failed
job is running. But there are probably also different reasons of such bad CI
state and I think that our next steps should be to think about what are the
main reasons of it and to try to find some ways to improve it in near future.

At the end I want also to thanks [Nate Johnston](https://natejohnston.info/) for
reviewing draft of this post :)
