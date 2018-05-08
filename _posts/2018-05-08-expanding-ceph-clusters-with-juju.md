---
title:  "Expanding Ceph clusters with Juju"
excerpt: "How to easily scale your Ceph cluster with Juju with minimal performance impact" 
categories:
  - posts
tags:
  - juju
  - ceph
toc: true
---

We just got a set of new SuperMicro servers for one of our Ceph clusters at [HUNT Cloud](https://www.ntnu.edu/huntgenes/hunt-cloud).
This made for a great opportunity to write up the simple steps of expanding a Ceph cluster with Juju.

New to Juju? [Juju](https://jujucharms.com) is a cool controller and agent based tool from Canonical to easily deploy and manage applications (called Charms) on different clouds and environments (see [how it works](https://jujucharms.com/how-it-works) for more details).

Scaling applications with Juju is easy and Ceph is no exception.
You can deploy more Ceph OSD hosts with just a simple `juju add-unit ceph-osd` command.
The challenging part is to add new OSDs without impacting client performance due to large amounts of backfilling.

Below is a brief walk through of the steps on how you can scale your Ceph cluster, with a small example cluster that you can deploy locally on LXD containers to follow along:

* Set crush initial weight to 0
* Add new OSDs to the cluster
* Clear crush initial weight
* Reweight new OSDs

## Deploy a Ceph cluster on LXD for testing

The first step is to get a Juju controller up and running so you can deploy Ceph.
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

Next up you need to deploy the Ceph cluster.
Here's a bundle called [`ceph-lxd`](https://jujucharms.com/u/szeestraten/ceph-lxd/bundle) which sets up a small cluster for you with:

* 1 Ceph Monitor host using the [`ceph-mon`](https://jujucharms.com/ceph-mon/) charm
* 3 Ceph OSD hosts with 3 OSDs each using the [`ceph-osd`](https://jujucharms.com/ceph-osd/) charm

You can deploy it straight from the Juju charm store:

```shell
$ juju deploy cs:~szeestraten/bundle/ceph-lxd

Located bundle "cs:~szeestraten/bundle/ceph-lxd-1"
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

The deployment may take a little while, so here's your perfect chance to refill your coffee.
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
ceph-mon/0*  active    idle   0        10.247.146.247         Unit is ready and clustered
ceph-osd/0*  active    idle   1        10.247.146.135         Unit is ready (3 OSD)
ceph-osd/1   active    idle   2        10.247.146.173         Unit is ready (3 OSD)
ceph-osd/2   active    idle   3        10.247.146.143         Unit is ready (3 OSD)

Machine  State    DNS             Inst id        Series  AZ  Message
0        started  10.247.146.247  juju-07321b-0  xenial      Running
1        started  10.247.146.135  juju-07321b-1  xenial      Running
2        started  10.247.146.173  juju-07321b-2  xenial      Running
3        started  10.247.146.143  juju-07321b-3  xenial      Running

Relation provider  Requirer      Interface  Type     Message
ceph-mon:mon       ceph-mon:mon  ceph       peer
ceph-mon:osd       ceph-osd:mon  ceph-osd   regular
```

Now, take a closer look at the cluster to ensure that it is in a healthy state and that all OSDs have been created:

```shell
$ juju ssh ceph-mon/0 "sudo -s"

$ ceph status
  cluster:
    id:     f719d3e8-52ff-11e8-91f4-00163e5622ff
    health: HEALTH_OK

  services:
    mon: 1 daemons, quorum juju-07321b-0
    mgr: juju-07321b-0(active)
    osd: 9 osds: 9 up, 9 in

  data:
    pools:   0 pools, 0 pgs
    objects: 0 objects, 0 bytes
    usage:   7367 MB used, 234 GB / 242 GB avail
    pgs:
```

The output above says that the cluster does not contain any pools, pgs or objects.
So, let's fix that by creating a pool called `testpool` and writing some data to it with one of Ceph's internal benchmarking tools, `rados bench`, so that we have something to actually shuffle around:

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

Let's check `ceph status` once again.
You should now see the new pool and some objects created by the benchmarking tool.

```shell
$ ceph status
  cluster:
    id:     f719d3e8-52ff-11e8-91f4-00163e5622ff
    health: HEALTH_OK

  services:
    mon: 1 daemons, quorum juju-07321b-0
    mgr: juju-07321b-0(active)
    osd: 9 osds: 9 up, 9 in

  data:
    pools:   1 pools, 100 pgs
    objects: 262 objects, 1044 MB
    usage:   7537 MB used, 234 GB / 241 GB avail
    pgs:     100 active+clean
```

Finally, let's take a look at `ceph osd tree` which prints out a tree of all the OSDs according to their position in the CRUSH map.
Pay particular attention to the `WEIGHT` column as we will be manipulating these values for the new OSDs when you expand the cluster later.

```shell
$ ceph osd tree
ID CLASS WEIGHT  TYPE NAME              STATUS REWEIGHT PRI-AFF
-1       0.23662 root default
-3       0.07887     host juju-07321b-1
 1   hdd 0.02629         osd.1              up  1.00000 1.00000
 3   hdd 0.02629         osd.3              up  1.00000 1.00000
 6   hdd 0.02629         osd.6              up  1.00000 1.00000
-5       0.07887     host juju-07321b-2
 0   hdd 0.02629         osd.0              up  1.00000 1.00000
 4   hdd 0.02629         osd.4              up  1.00000 1.00000
 8   hdd 0.02629         osd.8              up  1.00000 1.00000
-7       0.07887     host juju-07321b-3
 2   hdd 0.02629         osd.2              up  1.00000 1.00000
 5   hdd 0.02629         osd.5              up  1.00000 1.00000
 7   hdd 0.02629         osd.7              up  1.00000 1.00000
```

Don't worry if you don't end up with the exact weight or usage numbers as above.
Those numbers depend on the size of storage available on the LXD host.

So, with a working Ceph cluster, let's get start the expansion process.

## Set crush initial weight to 0

As mentioned at the top, the challenge is to manage the amount of backfilling when adding new OSDs.
One way of doing this is to make sure that all new OSDs get an initial weight of 0.
This ensures that Ceph doesn't start shuffling around data right away when we introduce the new OSDs.

The `ceph-osd` charm has a handy configuration option called [`crush-initial-weight`](https://jujucharms.com/ceph-osd/#charm-config-crush-initial-weight) which allows us to set this easily across all OSDs hosts:

```shell
juju config ceph-osd crush-initial-weight=0
```

> **Note:** There was a [bug](https://bugs.launchpad.net/charm-ceph-osd/+bug/1764077) in the [ceph-osd charm](https://jujucharms.com/ceph-osd/) before revision 261 which did not render the correct configuration when setting `crush-initial-weight=0`.
> Here's a workaround for those on older revisions:
> ```shell
> juju config ceph-osd config-flags='{ "global": { "osd crush initial weight": 0 } }'
> ```

## Add new OSDs to the cluster

Juju makes adding new Ceph OSD hosts a breeze.
You simply tell it to add another `ceph-osd` unit and it will take care of the rest.

```shell
juju add-unit ceph-osd
```

Time for another coffee while you wait for a new LXD container to spin up.
If all goes well, you should end up with a fourth `ceph-osd` unit in your list:

```shell
$ juju status

Model    Controller  Cloud/Region  Version  SLA
default  lxd         lxd           2.3.7    unsupported

App       Version  Status  Scale  Charm     Store       Rev  OS      Notes
ceph-mon  12.2.4   active      1  ceph-mon  jujucharms   24  ubuntu
ceph-osd  12.2.4   active      4  ceph-osd  jujucharms  261  ubuntu

Unit         Workload  Agent  Machine  Public address  Ports  Message
ceph-mon/0*  active    idle   0        10.247.146.247         Unit is ready and clustered
ceph-osd/0*  active    idle   1        10.247.146.135         Unit is ready (3 OSD)
ceph-osd/1   active    idle   2        10.247.146.173         Unit is ready (3 OSD)
ceph-osd/2   active    idle   3        10.247.146.143         Unit is ready (3 OSD)
ceph-osd/3   active    idle   4        10.247.146.230         Unit is ready (3 OSD)

Machine  State    DNS             Inst id        Series  AZ  Message
0        started  10.247.146.247  juju-07321b-0  xenial      Running
1        started  10.247.146.135  juju-07321b-1  xenial      Running
2        started  10.247.146.173  juju-07321b-2  xenial      Running
3        started  10.247.146.143  juju-07321b-3  xenial      Running
4        started  10.247.146.230  juju-07321b-4  xenial      Running

Relation provider  Requirer      Interface  Type     Message
ceph-mon:mon       ceph-mon:mon  ceph       peer
ceph-mon:osd       ceph-osd:mon  ceph-osd   regular
```

Note that the new host and OSDs should get a weight of 0 in the CRUSH tree (here represented by `juju-07321b-4` and `osd.9`, `osd.10` and `osd.11`).

```shell
$ ceph osd tree
ID CLASS WEIGHT  TYPE NAME              STATUS REWEIGHT PRI-AFF
-1       0.23662 root default
-3       0.07887     host juju-07321b-1
 1   hdd 0.02629         osd.1              up  1.00000 1.00000
 3   hdd 0.02629         osd.3              up  1.00000 1.00000
 6   hdd 0.02629         osd.6              up  1.00000 1.00000
-5       0.07887     host juju-07321b-2
 0   hdd 0.02629         osd.0              up  1.00000 1.00000
 4   hdd 0.02629         osd.4              up  1.00000 1.00000
 8   hdd 0.02629         osd.8              up  1.00000 1.00000
-7       0.07887     host juju-07321b-3
 2   hdd 0.02629         osd.2              up  1.00000 1.00000
 5   hdd 0.02629         osd.5              up  1.00000 1.00000
 7   hdd 0.02629         osd.7              up  1.00000 1.00000
-9             0     host juju-07321b-4
 9   hdd       0         osd.9              up  1.00000 1.00000
10   hdd       0         osd.10             up  1.00000 1.00000
11   hdd       0         osd.11             up  1.00000 1.00000
```

> If you only want to add OSDs (drives) to existing Ceph OSD hosts you can use the [`osd-devices`](https://jujucharms.com/ceph-osd/#charm-config-osd-devices) configuration option.
> Here's an example for this test cluster which adds a new directory as a new OSD for all `ceph-osd` hosts:
> ```shell
> juju config ceph-osd osd-devices="/srv/osd1 /srv/osd2 /srv/osd3 /srv/osd4"
> ```

## Clear crush initial weight

With all new OSDs in place, you can clear the `crush-initial-weight` configuration option we set earlier.

```shell
juju config ceph-osd --reset crush-initial-weight
```

## Reweight new OSDs

You now need to increase the weight of the new OSDs in the CRUSH map from 0 to their target weight in order for Ceph store data on them.

There are a couple of different ways to go depending on how many new OSDs you have and how carefully/slowly you want to introduce them:

* Reweight OSDs with [`ceph osd crush reweight <name> <weight>`](http://docs.ceph.com/docs/master/man/8/ceph/#commands)
  * Good for individual or small amounts of OSDs
* Reweight subtrees [`ceph osd crush reweight-subtree <name> <weight>`](http://docs.ceph.com/docs/master/man/8/ceph/#commands)
  * Good for larger amount of OSDs under a bucket (such as `host`, `chassis`, `rack` etc.)
* Gently reweight a list of OSDs with [`ceph-gentle-reweight`](https://github.com/cernceph/ceph-scripts/blob/master/tools/ceph-gentle-reweight), a tool from the folks at CERN
  * Good for gradually adding/removing a list of OSDs and you want limit disruption by monitoring latency and backfilling
* [Manually editing the CRUSH map](http://docs.ceph.com/docs/master/rados/operations/crush-map-edits/)
  * Good for manually controlling the CRUSH map with automation, version control etc.

The main point is that you want to increment the weight of the OSDs in small steps in order to control the amount of backfilling Ceph will have to do.
As with all things Ceph, this will of course depend on your cluster so I highly recommended you try these things in a staging cluster and start small.

For our test, let's use the `reweight-subtree` command with and weight of `0.01` so we can reweight all the OSDs of the new Ceph OSD host:

```shell
$ ceph osd crush reweight-subtree juju-07321b-4 0.01
reweighted subtree id -9 name 'juju-07321b-4' to 0.01 in crush map
```

While Ceph is working, keep an eye on the CRUSH tree and the disk usage with this command:

```shell
$ ceph osd df tree
ID CLASS WEIGHT  REWEIGHT SIZE   USE    AVAIL  %USE VAR  PGS TYPE NAME
-1       0.26660        -   316G 10012M   306G 3.09 1.00   - root default
-3       0.07887        - 80948M  2509M 78439M 3.10 1.00   -     host juju-07321b-1
 1   hdd 0.02629  1.00000 26982M   836M 26146M 3.10 1.00  26         osd.1
 3   hdd 0.02629  1.00000 26983M   836M 26146M 3.10 1.00  27         osd.3
 6   hdd 0.02629  1.00000 26983M   836M 26146M 3.10 1.00  31         osd.6
-5       0.07887        - 80947M  2507M 78439M 3.10 1.00   -     host juju-07321b-2
 0   hdd 0.02629  1.00000 26982M   835M 26146M 3.10 1.00  24         osd.0
 4   hdd 0.02629  1.00000 26982M   835M 26146M 3.10 1.00  24         osd.4
 8   hdd 0.02629  1.00000 26982M   835M 26146M 3.10 1.00  32         osd.8
-7       0.07887        - 80951M  2511M 78439M 3.10 1.00   -     host juju-07321b-3
 2   hdd 0.02629  1.00000 26983M   837M 26146M 3.10 1.00  20         osd.2
 5   hdd 0.02629  1.00000 26983M   837M 26146M 3.10 1.00  41         osd.5
 7   hdd 0.02629  1.00000 26983M   837M 26146M 3.10 1.00  29         osd.7
-9       0.02998        - 80923M  2483M 78439M 3.07 0.99   -     host juju-07321b-4
 9   hdd 0.00999  1.00000 26974M   827M 26146M 3.07 0.99  10         osd.9
10   hdd 0.00999  1.00000 26974M   827M 26146M 3.07 0.99  13         osd.10
11   hdd 0.00999  1.00000 26974M   827M 26146M 3.07 0.99  23         osd.11
                    TOTAL   316G 10012M   306G 3.09
MIN/MAX VAR: 0.99/1.00  STDDEV: 0.01
```

As you can see, the new weight of the new OSDs `osd.9`, `osd.10` and `osd.11` is `~0.01`.

Again, for real clusters, reweighting in multiple small steps is what will take most of the time and what you really want to automate.

To keep this short, let's reweight the OSDs again, this time directly to the target weight of the original OSDs:

```shell
$ ceph osd crush reweight-subtree juju-07321b-4 0.026299
reweighted subtree id -9 name 'juju-07321b-4' to 0.026299 in crush map

$ ceph osd df tree
ID CLASS WEIGHT  REWEIGHT SIZE   USE    AVAIL  %USE VAR  PGS TYPE NAME
-1       0.31549        -   316G 10029M   306G 3.10 1.00   - root default
-3       0.07887        - 80931M  2509M 78421M 3.10 1.00   -     host juju-07321b-1
 1   hdd 0.02629  1.00000 26977M   836M 26140M 3.10 1.00  25         osd.1
 3   hdd 0.02629  1.00000 26977M   836M 26140M 3.10 1.00  26         osd.3
 6   hdd 0.02629  1.00000 26977M   836M 26140M 3.10 1.00  25         osd.6
-5       0.07887        - 80927M  2505M 78421M 3.10 1.00   -     host juju-07321b-2
 0   hdd 0.02629  1.00000 26975M   835M 26140M 3.10 1.00  20         osd.0
 4   hdd 0.02629  1.00000 26975M   835M 26140M 3.10 1.00  23         osd.4
 8   hdd 0.02629  1.00000 26975M   835M 26140M 3.10 1.00  22         osd.8
-7       0.07887        - 80933M  2511M 78421M 3.10 1.00   -     host juju-07321b-3
 2   hdd 0.02629  1.00000 26977M   837M 26140M 3.10 1.00  21         osd.2
 5   hdd 0.02629  1.00000 26977M   837M 26140M 3.10 1.00  33         osd.5
 7   hdd 0.02629  1.00000 26977M   837M 26140M 3.10 1.00  27         osd.7
-9       0.07887        - 80925M  2503M 78421M 3.09 1.00   -     host juju-07321b-4
 9   hdd 0.02629  1.00000 26975M   834M 26140M 3.09 1.00  25         osd.9
10   hdd 0.02629  1.00000 26975M   834M 26140M 3.09 1.00  21         osd.10
11   hdd 0.02629  1.00000 26975M   834M 26140M 3.09 1.00  32         osd.11
                    TOTAL   316G 10029M   306G 3.10
MIN/MAX VAR: 1.00/1.00  STDDEV: 0.00
```

And there you go.
All the new OSDs have been introduced to the cluster and weighted correctly.

## Afterword

Thanks to all the [OpenStack Charmers](https://docs.openstack.org/charm-guide/latest/index.html) for creating and keeping all of these charms in great shape.
Also thanks to Dan van der Ster and the storage folks at CERN for the tools and many great tips on how to run Ceph at scale.
Finally, many thanks to Oddgeir Lingaas Holmen for helping write and clean up these posts.