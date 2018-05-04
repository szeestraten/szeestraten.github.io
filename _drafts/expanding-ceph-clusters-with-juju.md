---
title:  "Expanding Ceph clusters with Juju"
excerpt: "How to easily scale your Ceph cluster with Juju without impacting performance (too much)" 
categories:
  - posts
tags:
  - juju
  - ceph
---

I recently spent some time upgrading our Juju environments from 2.1 to 2.3. Below are a few lessons learned aimed at other Juju enthusiasts doing the same experiment.

## One way to expand your Ceph cluster (for dummies)

1. Set `osd crush initial weight = 0` config option in `ceph.conf` with `juju config ceph-osd crush-initial-weight=0`
2. Add new OSDs to the cluster `juju add-unit ceph-osd`
3. Clear `osd crush initial weight` with `juju config ceph-osd --reset crush-initial-weight`
4. Gently reweight new OSDs

## Deploy a minimal Ceph cluster

First, make sure you have a LXD/localhost cloud:

```shell
juju bootstrap localhost
```



```shell
juju deploy cs:~szeestraten/bundle/ceph-lxd-0
```

```yaml
description: "Minimal Ceph deployment for LXD"
relations:
- - ceph-osd:mon
  - ceph-mon:osd
series: xenial
services:
  ceph-mon:
    annotations:
      gui-x: '750'
      gui-y: '500'
    charm: cs:ceph-mon-24
    num_units: 1
    options:
      monitor-count: 1
      source: cloud:xenial-queens
  ceph-osd:
    annotations:
      gui-x: '1000'
      gui-y: '500'
    charm: cs:ceph-osd-261
    num_units: 3
    options:
      osd-devices: "/srv/osd"
      use-direct-io: False
      source: cloud:xenial-queens
```

Grab a coffee while Juju finishes up the deployment.

Verify Ceph has been deployed

```shell
juju status
```

```shell
Model    Controller  Cloud/Region  Version  SLA
default  lxd         lxd           2.3.7    unsupported

App       Version  Status  Scale  Charm     Store       Rev  OS      Notes
ceph-mon  12.2.4   active      1  ceph-mon  jujucharms   24  ubuntu
ceph-osd  12.2.4   active      3  ceph-osd  jujucharms  261  ubuntu

Unit         Workload  Agent  Machine  Public address  Ports  Message
ceph-mon/0*  active    idle   0        10.181.145.126         Unit is ready and clustered
ceph-osd/0*  active    idle   1        10.181.145.136         Unit is ready (1 OSD)
ceph-osd/1   active    idle   2        10.181.145.173         Unit is ready (1 OSD)
ceph-osd/2   active    idle   3        10.181.145.2           Unit is ready (1 OSD)

Machine  State    DNS             Inst id        Series  AZ  Message
0        started  10.181.145.126  juju-20c106-0  xenial      Running
1        started  10.181.145.136  juju-20c106-1  xenial      Running
2        started  10.181.145.173  juju-20c106-2  xenial      Running
3        started  10.181.145.2    juju-20c106-3  xenial      Running

Relation provider  Requirer      Interface  Type     Message
ceph-mon:mon       ceph-mon:mon  ceph       peer
ceph-mon:osd       ceph-osd:mon  ceph-osd   regular
```

Check current CRUSH Map

```shell
juju ssh ceph-mon/0 "sudo -s"
```

```shell
ceph osd df tree
```

```shell
ID CLASS WEIGHT  REWEIGHT SIZE   USE   AVAIL  %USE VAR  PGS TYPE NAME
-1       0.07860        - 82539M 2455M 80084M 2.97 1.00   - root default
-7       0.02620        - 27513M  818M 26694M 2.97 1.00   -     host juju-20c106-1
 1   hdd 0.02620  1.00000 27513M  818M 26694M 2.97 1.00   0         osd.1
-5       0.02620        - 27513M  818M 26694M 2.97 1.00   -     host juju-20c106-2
 2   hdd 0.02620  1.00000 27513M  818M 26694M 2.97 1.00   0         osd.2
-3       0.02620        - 27513M  818M 26694M 2.97 1.00   -     host juju-20c106-3
 0   hdd 0.02620  1.00000 27513M  818M 26694M 2.97 1.00   0         osd.0
                    TOTAL 82539M 2455M 80084M 2.97
MIN/MAX VAR: 1.00/1.00  STDDEV: 0
```

## Set crush initial weight to 0

1. Set `osd crush initial weight = 0` in `ceph.conf`

[crush-initial-weight](https://jujucharms.com/ceph-osd/#charm-config-crush-initial-weight) config option

```shell
juju config ceph-osd crush-initial-weight=0
```

> **Note:** There was a [bug](https://bugs.launchpad.net/charm-ceph-osd/+bug/1764077) in the [ceph-osd charm](https://jujucharms.com/ceph-osd/) before revision 261 which did not render the correct configuration when setting `crush-initial-weight` option to 0.
> Here's a workaround for those on older revisions:
>```shell
> juju config ceph-osd config-flags='{ "global": { "osd crush initial weight": 0 } }'
>```

## Expand Ceph cluster

```shell
juju add-unit ceph-osd
```

or

```shell
juju config ceph-osd osd-devices="/srv/osd /srv/newosd"
```

## Reset crush initial weight

```shell
juju config ceph-osd --reset crush-initial-weight
```

## Reweight OSDs

```shell
ceph osd crush reweight-subtree <device> <weight>
```

```shell
ceph osd crush reweight-subtree hcc-dev12 2
```

Keep an eye on the OSD tree and the disk usage.

```shell
ceph osd df tree
```

[ceph-gentle-reweight](https://github.com/cernceph/ceph-scripts/blob/master/tools/ceph-gentle-reweight)

https://indico.cern.ch/event/567516/contributions/2293565/attachments/1331568/2001323/050916-cephupdate.pdf


Enjoy a beverage.