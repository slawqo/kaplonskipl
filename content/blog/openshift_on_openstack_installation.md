---
title: "Installation of Openshift on the OpenStack cloud"
date: 2020-11-07T17:02:52+01:00
tags: ["openstack", "openshift"]
draft: false
---

# Installation of Openshift on the OpenStack cloud

During last "Day of Learning" in [Red
Hat](https://redhat.com) which was great opportunity for me to spent whole day
on learning something new to me and I choose to learn a bit about installation
and management of the Openshift cluster.
This post is mostly note for myself from what I did during that training.
I was using Openshift 4.6.1 and I installed it on the OpenStack based cloud.

## Prerequisites

To install Openshift on OpenStack cloud You need to prepare few things.

### clouds.yaml
This is file used by openshift installer (and OpenStack client too) to get
credentials to the cloud.
It can be downloaded from Horizon: _Project -> API Access ->Download OpenStack
RC File -> OpenStack clouds.yaml File_.
This file should be located in ~/.config/openstack/clouds.yaml on Your computer.

### SSH key
You need to have ssh key without pass phrase. You can generate it with command:

```
ssh-keygen -t rsa -b 4096
```
and follow intructions displayed there.

### OpenStack preparation
You need to have external network (at least in the base variant of the
installation which I was doing) and 2 Floating IPs which will be used as API IP
and IP for apps endpoint.
You also need to have flavor with at least 4 vCPU.
Openshift installer will ask You about those things.

### DNS
Installer will try to connect to the openshift cluster using domain name so You
should have domains:

```
api.<cluster_name>.<base_domain>.  IN  A  <API_FIP_IP>
*.apps.<cluster_name>.<base_domain>. IN  A <APPS_FIP_IP>
```
For the development or testing purpose You can set it in some local dns server,
like e.g. dnsmasq if You don't control DNS entries for used domain.

### Openshift secrerts

You need to pull secrets. It can be downloaded from https://try.openshift.com
where You should login with Your Red Hat Developer account.

## Install config file
You can run installer and give all requested data in the interactive shell. But
that isn't very efficient if You are spawning clusters many times.  So You can
also create file __install-config.yaml__ which will contain all information
required by the installer.

```
$ cat ./my-first-cluster/install-config.yaml
apiVersion: v1
baseDomain: "skaplons.cluster"
clusterID:  "7060401a-60bc-4c49-b6c8-b76f8f18b580"
compute:
- name: worker
  platform: {}
  replicas: 1
controlPlane:
  name: master
  platform: {}
  replicas: 3
metadata:
  name: "skaplons"
networking:
  clusterNetworks:
  - cidr: 10.128.0.0/14
    hostSubnetLength: 9
  serviceCIDR: 172.30.0.0/16
  machineCIDR: 10.196.0.0/16
platform:
  openstack:
    cloud:              "upshift"
    externalNetwork:    "provider_net_shared"
    region:             "regionOne"
    computeFlavor:      "ocp-master"
    lbFloatingIP:       "<API_FIP_IP>"
    ingressFloatingIp:  "<APPS_FIP_ID>"
pullSecret: 'pull secret from try.openshift.com should be here'
sshKey: 'public ssh key should be here'
```

## Installation
When all that is ready You can download and run installer:

```
$ wget https://mirror.openshift.com/pub/openshift-v4/clients/ocp/latest/openshift-install-linux.tar.gz
$ ./openshift-install create cluster --dir=./my-first-cluster
```

Now You can get some coffee and wait - installation should take about 30-40
minutes.

## Cluster verification

To interact with Openshift cluster You need to have tool called __oc__ which can
be downloaded from [cloud.redhat.com](https://cloud.redhat.com):

```
$ wget https://mirror.openshift.com/pub/openshift-v4/clients/ocp/latest/openshift-client-linux.tar.gz
$ tar zxfv openshift-client-linux.tar.gz
```

Now You should be able to interact with new cluster
```
$ export KUBECONFIG=`pwd`/my-first-cluster/auth/kubeconfig
$ ./oc whoami
system:admin
./oc get nodes                                                                                                                                                                                                                                                        130 ↵
NAME                            STATUS   ROLES    AGE   VERSION
skaplons-mptzf-master-0         Ready    master   46m   v1.19.0+d59ce34
skaplons-mptzf-master-1         Ready    master   47m   v1.19.0+d59ce34
skaplons-mptzf-master-2         Ready    master   46m   v1.19.0+d59ce34
skaplons-mptzf-worker-0-dbfb5   Ready    worker   34m   v1.19.0+d59ce34
```

You can also get link to the web console
```
$ oc whoami --show-console
```

## Cluster management
Now, when cluster is installed You can manage it. Nodes in openshift cluster are
managed by _machine_ API. To list nodes You can run command:

```
$ ./oc get machines -n openshift-machine-api

NAME                            PHASE     TYPE         REGION      ZONE   AGE
skaplons-mptzf-master-0         Running   ocp-master   regionOne   nova   80m
skaplons-mptzf-master-1         Running   ocp-master   regionOne   nova   80m
skaplons-mptzf-master-2         Running   ocp-master   regionOne   nova   80m
skaplons-mptzf-worker-0-dbfb5   Running   ocp-master   regionOne   nova   78m
```

You can also get much more details about specific node:
```
$ ./oc describe machine skaplons-mptzf-worker-0 -n openshift-machine-api
<cut output here>
```

Of course You can do much more like listing all cluster control plane, or list
region name of the nodes etc. but I will not list all possible commands here.
For example to list all nodes and their flavor You can use command:

```
$ ./oc get machines -n openshift-machine-api -o jsonpath='{range .items[*]}{"\n"}{.metadata.name}{"\t"}{.spec.providerSpec.value.flavor}{end}{"\n"}'

skaplons-mptzf-master-0 ocp-master
skaplons-mptzf-master-1 ocp-master
skaplons-mptzf-master-2 ocp-master
skaplons-mptzf-worker-0-dbfb5   ocp-master
```

Machines can be grouped in _machinesets_ which can be listed with command:
```
$ ./os get machinesets -n openshift-machine-api
NAME                      DESIRED   CURRENT   READY   AVAILABLE   AGE
skaplons-mptzf-worker-0   1
```

### Scalling workers in machineset
You can scale up/down nodes in the machineset with command like:

```
$ ./oc scale machineset skaplons-mptzf-worker-0 --replicas=2 -n openshift-machine-api
```

If You are scalling nodes down to e.g. 0 workes, worker may not be really
deleted but stay in _Deleting_ state until You will add other workers e.g. in
different region becuase existing pods needs to be moved somewhere before worker
will be deleted.

Using scalling You can also change flavor of workers used in the machineset. To
do that You need to execute commands like

```
$ ./oc patch machineset skaplons-mptzf-worker-0 --type='merge' --patch='{"spec":{"template": {"spec": {"providerSpec": {"value": { "flavor": "ci.m5.large"}}}}}}' -n openshift-machine-api
machineset.machine.openshift.io/skaplons-mptzf-worker-0 patched
```

and after that if You scale up Your machineset, new nodes will be using new
flavor. Existing nodes will still use old flavor.

## Summary
Installation of the Openshift cluster is really very easy. Of course, You can
use some public cloud providers, like AWS or Google Cloud instead of OpenStack
base cloud (but why would You really? :)). Openshift can be also installed
directly on bare metal but I didn't try that way at all.
As a next steps I will probably explore Openshift a bit more and learn more
details internals of it. But for now I think it's enough for one day of learning
:)

