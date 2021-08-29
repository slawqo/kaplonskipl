---
title: "Using eBPF to trace OpenStack services"
date: 2021-08-29T18:25:25+02:00
tags: ["openstack"]
draft: false
---

## eBPF - what is it?

eBPF stands for extended Berkeley Packet Filter and it allows write and run eBPF
programs in the virtual machine inside the Linux kernel. More about it can be
found for example on the [Brendan Gregg's
blog](https://www.brendangregg.com/blog/2019-01-01/learn-ebpf-tracing.html). I
used it to learn eBPF basics and I think it's very good place to start really.
eBPF is the "engine" which allows to run programs but there are also other tools
(frameworks) which allows to write eBPF programs in the easier way. Such tools
are **BCC** and **bpftrace**.

## BCC tools

### Installation

Installation guide for the ``bcc-tools`` is available in the [git
repo](https://github.com/iovisor/bcc/blob/master/INSTALL.md). I also prepared
for myself small [ansible role](https://github.com/slawqo/ansible/tree/master/roles/ebpf-bcc-tracing)
which can install it on the Centos 8 and Ubuntu machines.

### Existing programs
BCC already have many programs available - all can be found in the [iovisor git
repo](https://github.com/iovisor/bcc/tree/master/tools). For each program there
is also ``\_example.txt`` file there. That ``\_example.txt`` file contains some
examples how each program can be used and what it is doing.
In this post I want to focus on how to install and use `bcc-tools` to trace
performance of the OpenStack Neutron processes.

## Use BCC tools to analyze openstack neutron performance

Now as bcc-tools are installed, let's try to use some of them to trace
performance of the OpenStack Neutron processes.
I run some small set of scenario tests from neutron-tempest-plugin to have some
load on my Devstack machine while I was using bcc tools.

### DB queries
BCC have tools like
[``dbstat``](https://github.com/iovisor/bcc/blob/master/tools/dbstat_example.txt)
and
[``dbslower``](https://github.com/iovisor/bcc/blob/master/tools/dbslower_example.txt)
which may be used to understand what is going on with our database.

* dbstat
According to official documentation "dbstat traces queries performed by a MySQL
or PostgreSQL database process, and displays a histogram of query latencies".
Example output can be like:
```
# /usr/share/bcc/tools/dbstat mysql
Tracing database queries for pids  slower than 0 ms...
^C
     query latency (ms)  : count     distribution
         0 -> 1          : 163036   |****************************************|
         2 -> 3          : 1162     |                                        |
         4 -> 7          : 901      |                                        |
         8 -> 15         : 377      |                                        |
        16 -> 31         : 96       |                                        |
        32 -> 63         : 12       |                                        |
        64 -> 127        : 1        |                                        |

```
As a result we have got histogram of db queries time. From above example we can
see that there were no really slow queries there. All were performed pretty
fast.

* dbslower
This tool can show db queries slower than given treshold. Treshold is 1 ms by
default, but it can be changed with `-m [treshold]ms`. Output in my case:

```
# /usr/share/bcc/tools/dbslower mysql -m 50
Tracing database queries for pids  slower than 50 ms...
TIME(s)        PID          MS QUERY
19.241605      123672   58.733 b'UPDATE agents SET heartbeat_timestamp=\'2021-08-27 12:23:15.641771\', configurations=\'{\\"dhcp_driver\\": \\"neutron.agent.linux.dhcp.Dnsmasq\\", \\"dhcp_lease_duration\\": 86400, \\"log_agent_heartbeats\\": false, \\"networks\\": 4, \\"ports\\": 8, \\"subnets\\": 5}\', `l'
187.098090     123672   74.499 b"UPDATE standardattributes SET revision_number=5, updated_at='2021-08-27 12:26:03' WHERE standardattributes.id = 501 AND standardattributes.revision_number = 4"

```

Here I wanted to see queries which were slower than 50 ms. There were only 2
such queries in the time when I was running my tests. Output shows timestamp of
when query was made, PID of the db process where query was done - in my case it
was PID of the ``mysqld`` process, time in miliseconds and query.

### Tracing python processes

BCC tools can be also used to trace python processes (also programs written in
different languages but I will focus on Python only). We can for example do
things like:

* check garbage collector events in the Python process -
  [``pytongc``](https://github.com/iovisor/bcc/blob/master/tools/lib/ugc_example.txt)

```
# /usr/share/bcc/tools/pythongc 156539
Tracing garbage collections in python process 156539... Ctrl-C to quit.
START    TIME (us) DESCRIPTION
10.994   248.29   None
```

Above shows gc events for one of mine neutron-server processes (PID 156539)

* [``pythoncalls``](https://github.com/iovisor/bcc/blob/master/tools/lib/ucalls_example.txt)
  can show summarized method calls in the Python program (result is displayed
  after pythoncalls program is finished)

```
# /usr/share/bcc/tools/pythoncalls 156539
Tracing calls in process 156539 (language: python)... Ctrl-C to quit.
^C
METHOD                                              # CALLS
[...]
/usr/lib64/python3.6/logging/__init__.py.debug            6
/usr/local/lib/python3.6/site-packages/kombu/connection.py.connected        6
/usr/local/lib/python3.6/site-packages/kombu/connection.py.connection        6
/usr/lib64/python3.6/logging/__init__.py.getEffectiveLevel        6
/usr/local/lib/python3.6/site-packages/amqp/connection.py.connected        6
/usr/local/lib/python3.6/site-packages/eventlet/greenio/base.py.recv        7
/usr/local/lib/python3.6/site-packages/eventlet/greenio/base.py._recv_loop        7
/usr/local/lib/python3.6/site-packages/eventlet/hubs/poll.py.register        8
/usr/local/lib/python3.6/site-packages/amqp/transport.py._read        9
/usr/local/lib/python3.6/site-packages/eventlet/greenio/base.py.gettimeout        9
/usr/local/lib/python3.6/site-packages/amqp/connection.py.transport       10
/usr/local/lib/python3.6/site-packages/eventlet/greenio/base.py.settimeout       10
/usr/local/lib/python3.6/site-packages/amqp/transport.py.having_timeout       10
/usr/local/lib/python3.6/site-packages/kombu/connection.py.transport       15
/usr/local/lib/python3.6/site-packages/eventlet/greenthread.py.sleep      120
/usr/local/lib/python3.6/site-packages/eventlet/green/os.py.waitpid      120
/usr/local/lib/python3.6/site-packages/oslo_service/service.py._wait_child      120
/usr/local/lib/python3.6/site-packages/eventlet/hubs/hub.py.fire_timers      130
/usr/local/lib/python3.6/site-packages/eventlet/hubs/poll.py.wait      130
/usr/local/lib/python3.6/site-packages/eventlet/hubs/epolls.py.do_poll      130
/usr/local/lib/python3.6/site-packages/eventlet/hubs/hub.py.sleep_until      130
/usr/local/lib/python3.6/site-packages/eventlet/hubs/hub.py.prepare_timers      260
/usr/local/lib/python3.6/site-packages/eventlet/timeout.py.start      360
/usr/local/lib/python3.6/site-packages/eventlet/timeout.py.pending      360
[...]

```
Output of this program can be very long so I shown here only some part of it.

* tracing new threads - [``threadsnoop``](https://github.com/iovisor/bcc/blob/master/tools/threadsnoop_example.txt)

```
# /usr/share/bcc/tools/threadsnoop  -T                                                                                                                                                                                                    [64/2141]
TIME(ms)   PID    COMM             FUNC
0          228885 b'beam.smp'      b'[unknown]'
[...]
933        228914 b'qemu-img'      b'[unknown]'
[...]
3730       89438  b'ovs-vswitchd'  b'handle_fildes_io'
3892       170203 b'rpc-worker'    b'[unknown]'
[...]
```
Output of it can also be very long so I just shown some small part of it.

### Tracing TCP connections

BCC can be also used to track TCP connections on the host. For example:

* print every new active TCP connection (established via connect()) with
  [``tcpconnect``](https://github.com/iovisor/bcc/blob/master/tools/tcpconnect_example.txt)

```# /usr/share/bcc/tools/tcpconnect
Tracing connect ... Hit Ctrl-C to end
PID    COMM         IP SADDR            DADDR            DPORT
222451 python3.6    4  10.100.0.20      10.100.0.20      80
222450 python3.6    4  10.100.0.20      10.100.0.20      80
84293  1_scheduler  4  127.0.0.1        127.0.1.1        4369
```

It can be also used to track DNS responses and associate them to the
connections:
```
# /usr/share/bcc/tools/tcpconnect -d
Tracing connect ... Hit Ctrl-C to end
PID    COMM         IP SADDR            DADDR            DPORT  QUERY
84293  1_scheduler  4  127.0.0.1        127.0.1.1        4369   No DNS Query
231942 curl         4  192.168.121.196  216.58.201.78    80     google.com
```

There are also other tools like ``tcpaccept``, ``tcpdrop``, ``tcpretrans`` and
others to track TCP connections in other states.

## bpftrace

[``Bpftrace``](https://github.com/iovisor/bpftrace) is another, high level
tracing language for ``eBPF``.  As it's high level language, it's much easier to
write own eBPF programs using it, comparing to e.g. ``bcc``. You can learn more
about it from very good [bpftrace One-Liners
Tutorial](https://github.com/iovisor/bpftrace/blob/master/docs/tutorial_one_liners.md)
made by Brendan Gregg.
