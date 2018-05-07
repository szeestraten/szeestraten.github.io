---
title:  "So You Think You Can Expand Your Ceph Cluster With Juju"
excerpt: "How to scale your Ceph cluster with Juju (without impacting performance too much)" 
categories:
  - posts
tags:
  - juju
  - ceph
toc: true
---

So you think you can expand your Ceph cluster with Juju?
Scaling things with Juju is easy, so it should just be a simple `juju add-unit` command away right?

We recently got some new SuperMicro servers at work for our Ceph clusters which powers the storage behind [HUNT Cloud](https://www.ntnu.edu/huntgenes/hunt-cloud).

Adding new Ceph OSDs can be quite disruptive and impact client performance due to backfilling.

...

## How to expand a Ceph cluster

...

* Set crush initial weight to 0
* Add new OSDs to the cluster
* Clear crush initial weight
* Reweight new OSDs

Below is a brief walk through of the different steps on a small test cluster which you can deploy locally on LXD containers.

## Deploy a test cluster on LXD

First, we need a Juju controller running on your local machine before we can deploy Ceph.
If you're new to Juju and LXD, you can get started with the official docs [here](https://jujucharms.com/docs/stable/tut-lxd).
In case you already have installed all the requirements, you can simply bootstrap a new controller like so:

```shell
$ juju bootstrap localhost

Creating Juju controller "localhost-localhost" on localhost/localhost
Looking for packaged Juju agent version 2.3.7 for amd64
To configure your system to better support LXD containers, please see: https://github.com/lxc/lxd/blob/master/doc/production-setup.md
Launching controller instance(s) on localhost/localhost...
 - juju-a24b9d-0 (arch=amd64)
Installing Juju agent on bootstrap instance
Fetching Juju GUI 2.12.1
Waiting for address
Attempting to connect to 10.181.145.171:22
Connected to 10.181.145.171
Running machine configuration script...
Bootstrap agent now started
Contacting Juju controller at 10.181.145.171 to verify accessibility...
Bootstrap complete, "localhost-localhost" controller now available
Controller machines are in the "controller" model
Initial model "default" added
```

Next up we need to deploy a Ceph cluster.
I created a bundle called `ceph-lxd` which sets up a small cluster with:

* 1 Ceph Monitor host using the [`ceph-mon`](https://jujucharms.com/ceph-mon/) charm
* 3 Ceph OSD hosts with 3 OSDs each using the [`ceph-osd`](https://jujucharms.com/ceph-osd/) charm

You can deploy it straight from the Juju charm store:

```shell
$ juju deploy cs:~szeestraten/bundle/ceph-lxd

Located bundle "cs:~szeestraten/bundle/ceph-lxd-0"
Resolving charm: cs:ceph-mon-24
Resolving charm: cs:ceph-osd-261
Executing changes:
- upload charm cs:ceph-mon-24 for series xenial
- deploy application ceph-mon on xenial using cs:ceph-mon-24
- set annotations for ceph-mon
- upload charm cs:ceph-osd-261 for series xenial
- deploy application ceph-osd on xenial using cs:ceph-osd-261
- set annotations for ceph-osd
- add relation ceph-osd:mon - ceph-mon:osd
- add unit ceph-mon/0 to new machine 0
- add unit ceph-osd/0 to new machine 1
- add unit ceph-osd/1 to new machine 2
- add unit ceph-osd/2 to new machine 3
Deploy of bundle completed.
```

> If you want to take a closer look at the bundle, you can find it in the Juju charm store [`here`](https://jujucharms.com/u/szeestraten/ceph-lxd/bundle).

The deployment may take a while so grab a coffee.
You can follow along with `juju status` (or `watch --color juju status --color` in case you get impatient).
If all goes well, you should end up with something that looks like this:

```shell
$ juju status

Model    Controller  Cloud/Region  Version  SLA
default  lxd         lxd           2.3.7    unsupported

App       Version  Status  Scale  Charm     Store       Rev  OS      Notes
ceph-mon  12.2.4   active      1  ceph-mon  jujucharms   24  ubuntu
ceph-osd  12.2.4   active      3  ceph-osd  jujucharms  261  ubuntu

Unit         Workload  Agent  Machine  Public address  Ports  Message
ceph-mon/0*  active    idle   0        10.247.146.238         Unit is ready and clustered
ceph-osd/0*  active    idle   1        10.247.146.107         Unit is ready (3 OSD)
ceph-osd/1   active    idle   2        10.247.146.223         Unit is ready (3 OSD)
ceph-osd/2   active    idle   3        10.247.146.48          Unit is ready (3 OSD)

Machine  State    DNS             Inst id        Series  AZ  Message
0        started  10.247.146.238  juju-a95c97-0  xenial      Running
1        started  10.247.146.107  juju-a95c97-1  xenial      Running
2        started  10.247.146.223  juju-a95c97-2  xenial      Running
3        started  10.247.146.48   juju-a95c97-3  xenial      Running

Relation provider  Requirer      Interface  Type     Message
ceph-mon:mon       ceph-mon:mon  ceph       peer
ceph-mon:osd       ceph-osd:mon  ceph-osd   regular
```

Make sure to take a closer look at the cluster to make sure it is in a healthy state and created all the OSDs:

```shell
$ juju ssh ceph-mon/0 "sudo -s"

$ ceph status
  cluster:
    id:     51770576-5225-11e8-ad8a-00163e3d069f
    health: HEALTH_OK

  services:
    mon: 1 daemons, quorum juju-a95c97-0
    mgr: juju-a95c97-0(active)
    osd: 9 osds: 9 up, 9 in

  data:
    pools:   0 pools, 0 pgs
    objects: 0 objects, 0 bytes
    usage:   7367 MB used, 234 GB / 241 GB avail
    pgs:
```

As you might have noticed from the output above, the cluster does not contain any pools, pgs or objects.
Let's fix that by creating a pool called `testpool` and writing some data to it with one of Ceph's internal benchmarking tools, `rados bench`:

```shell
$ ceph osd pool create testpool 100 100
pool 'testpool' created

$ ceph osd pool application enable testpool rgw
enabled application 'rgw' on pool 'testpool'

$ rados bench -p testpool 10 write --no-cleanup
hints = 1
Maintaining 16 concurrent writes of 4194304 bytes to objects of size 4194304 for up to 10 seconds or 0 objects
Object prefix: benchmark_data_juju-47966f-0_25611
  sec Cur ops   started  finished  avg MB/s  cur MB/s last lat(s)  avg lat(s)
    0       0         0         0         0         0           -           0
    1      16        36        20    79.891        80    0.934116    0.444238
    2      16        63        47   93.8659       108     1.39832    0.513196
    3      16        80        64   85.2469        68     1.32455    0.610896
    4      16       107        91   90.9015       108    0.877032    0.653832
    5      16       125       109   87.1202        72    0.218025    0.675221
    6      16       147       131   87.2638        88    0.754443     0.66334
    7      16       170       154   87.9318        92    0.645512    0.689901
    8      16       204       188   93.9318       136    0.987254    0.663071
    9      16       230       214   95.0482       104     1.37409    0.645098
   10      16       259       243   97.1393       116     1.39946    0.630096
Total time run:         10.306007
Total writes made:      260
Write size:             4194304
Object size:            4194304
Bandwidth (MB/sec):     100.912
Stddev Bandwidth:       21.1702
Max bandwidth (MB/sec): 136
Min bandwidth (MB/sec): 68
Average IOPS:           25
Stddev IOPS:            5
Max IOPS:               34
Min IOPS:               17
Average Latency(s):     0.630013
Stddev Latency(s):      0.382552
Max latency(s):         2.26802
Min latency(s):         0.0696069
```

If you check `ceph status` again, you should see the new pool and some objects created by the benchmarking tool.

```shell
$ ceph status
  cluster:
    id:     51770576-5225-11e8-ad8a-00163e3d069f
    health: HEALTH_OK

  services:
    mon: 1 daemons, quorum juju-a95c97-0
    mgr: juju-a95c97-0(active)
    osd: 9 osds: 9 up, 9 in

  data:
    pools:   1 pools, 100 pgs
    objects: 112 objects, 444 MB
    usage:   7445 MB used, 234 GB / 241 GB avail
    pgs:     100 active+clean
```

Finally, we should take a look at `ceph osd tree` which prints out a tree of all the OSDs with some info according to their position in the CRUSH map.
This is handy as we will be adding new OSDs to the CRUSH map and manipulating their weight when expanding the cluster later.

```shell
$ ceph osd tree
ID CLASS WEIGHT  TYPE NAME              STATUS REWEIGHT PRI-AFF
-1       0.23579 root default
-3       0.07860     host juju-a95c97-1
 0   hdd 0.02620         osd.0              up  1.00000 1.00000
 3   hdd 0.02620         osd.3              up  1.00000 1.00000
 5   hdd 0.02620         osd.5              up  1.00000 1.00000
-5       0.07860     host juju-a95c97-2
 1   hdd 0.02620         osd.1              up  1.00000 1.00000
 4   hdd 0.02620         osd.4              up  1.00000 1.00000
 8   hdd 0.02620         osd.8              up  1.00000 1.00000
-7       0.07860     host juju-a95c97-3
 2   hdd 0.02620         osd.2              up  1.00000 1.00000
 6   hdd 0.02620         osd.6              up  1.00000 1.00000
 7   hdd 0.02620         osd.7              up  1.00000 1.00000
```

> Don't worry if you don't end up with the exact weight or usage numbers as above, it depends on the size of storage available on the LXD host.

## Set crush initial weight to 0

As mentioned in the beginning, we want to make sure that all new OSDs get an initial weight of 0 to ensure Ceph doesn't start shuffling around data.
The `ceph-osd` charm has a configuration option called [`crush-initial-weight`](https://jujucharms.com/ceph-osd/#charm-config-crush-initial-weight) which allows us to set this easily across all OSDs hosts:

```shell
juju config ceph-osd crush-initial-weight=0
```

> **Note:** There was a [bug](https://bugs.launchpad.net/charm-ceph-osd/+bug/1764077) in the [ceph-osd charm](https://jujucharms.com/ceph-osd/) before revision 261 which did not render the correct configuration when setting `crush-initial-weight=0`.
> Here's a workaround for those on older revisions:
> ```shell
> juju config ceph-osd config-flags='{ "global": { "osd crush initial weight": 0 } }'
> ```

## Add new OSDs to the cluster

Now that we have a functioning cluster ready for expansion, we can finally deploy new Ceph OSD hosts.
Juju makes this a breeze so we simply tell it to add another `ceph-osd` unit and it will take care of the rest.

```shell
juju add-unit ceph-osd
```

Time for another coffee while we wait for a new LXD container to spin up.
If all goes well, you should end up with another `ceph-osd` unit:

```shell
$ juju status

Model    Controller  Cloud/Region  Version  SLA
default  lxd         lxd           2.3.7    unsupported

App       Version  Status  Scale  Charm     Store       Rev  OS      Notes
ceph-mon  12.2.4   active      1  ceph-mon  jujucharms   24  ubuntu
ceph-osd  12.2.4   active      4  ceph-osd  jujucharms  261  ubuntu

Unit         Workload  Agent  Machine  Public address  Ports  Message
ceph-mon/0*  active    idle   0        10.181.145.141         Unit is ready and clustered
ceph-osd/0*  active    idle   1        10.181.145.78          Unit is ready (1 OSD)
ceph-osd/1   active    idle   2        10.181.145.131         Unit is ready (1 OSD)
ceph-osd/2   active    idle   3        10.181.145.209         Unit is ready (1 OSD)
ceph-osd/3   active    idle   4        10.181.145.225         Unit is ready (1 OSD)

Machine  State    DNS             Inst id        Series  AZ  Message
0        started  10.181.145.141  juju-459940-0  xenial      Running
1        started  10.181.145.78   juju-459940-1  xenial      Running
2        started  10.181.145.131  juju-459940-2  xenial      Running
3        started  10.181.145.209  juju-459940-3  xenial      Running
4        started  10.181.145.225  juju-459940-4  xenial      Running

Relation provider  Requirer      Interface  Type     Message
ceph-mon:mon       ceph-mon:mon  ceph       peer
ceph-mon:osd       ceph-osd:mon  ceph-osd   regular
```

The CRUSH tree should now contain the new host and OSD with a weight of 0, here represented by `juju-459940-4` and `osd.3`.

```shell
$ ceph osd tree
ID CLASS WEIGHT  TYPE NAME              STATUS REWEIGHT PRI-AFF
-1       0.07887 root default
-3       0.02629     host juju-459940-1
 0   hdd 0.02629         osd.0              up  1.00000 1.00000
-7       0.02629     host juju-459940-2
 2   hdd 0.02629         osd.2              up  1.00000 1.00000
-5       0.02629     host juju-459940-3
 1   hdd 0.02629         osd.1              up  1.00000 1.00000
-9             0     host juju-459940-4
 3   hdd       0         osd.3              up  1.00000 1.00000
```

> If you only want to add OSDs (drives) to existing Ceph OSD hosts you can use the [`osd-devices`](https://jujucharms.com/ceph-osd/#charm-config-osd-devices) config option.
> Here's an example for this test cluster which adds a new directory as a new OSD for all `ceph-osd` hosts:
> ```shell
> juju config ceph-osd osd-devices="/srv/osd /srv/newosd"
> ```

## Clear crush initial weight

When all of the new OSDs are in place, we can clear the `crush-initial-weight` configuration option.

```shell
juju config ceph-osd --reset crush-initial-weight
```

## Reweight new OSDs

In order for Ceph to store data on the new OSDs, we need to need increase their weight in the CRUSH map.
There are a couple of different ways to go depending on how many new OSDs there are and how slowly you want to introduce these new OSDs:

* Reweight OSDs with [`ceph osd crush reweight <name> <weight>`](http://docs.ceph.com/docs/master/man/8/ceph/#commands)
  * Good for individual or small amounts of OSDs
* Reweight subtrees [`ceph osd crush reweight-subtree <name> <weight>`](http://docs.ceph.com/docs/master/man/8/ceph/#commands)
  * Good for larger amount of OSDs under a bucket (such as `host`, `chassis`, `rack` etc.)
* Gently reweight a list of OSDs with [`ceph-gentle-reweight`](https://github.com/cernceph/ceph-scripts/blob/master/tools/ceph-gentle-reweight), a tool from the folks at CERN
  * Good for gradually adding/removing a list of OSDs and you want limit disruption by monitoring latency and backfilling
* [Manually editing the CRUSH map](http://docs.ceph.com/docs/master/rados/operations/crush-map-edits/)
  * Good for manually controlling the CRUSH map with automation, version control etc.

```shell
ceph osd crush reweight-subtree juju-459940-4 0.02629
```

Keep an eye on the CRUSH tree and the disk usage.

```shell
ceph osd df tree
```

## Afterword

Thanks to all the [OpenStack Charmers](https://docs.openstack.org/charm-guide/latest/index.html) for creating and keeping all of these charms in great shape.
Also thanks to Dan van der Ster and the storage folks at CERN for the tools and many great tips on how to run Ceph at scale.