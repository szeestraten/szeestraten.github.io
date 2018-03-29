---
title:  "Upgrading Juju"
excerpt: "Experiences upgrading highly available Juju environments from 2.1 to 2.3" 
categories:
  - posts
tags:
  - juju
---

I recently spent some time upgrading our Juju environments from 2.1 to 2.3. Below are a few lessons learned aimed at other Juju enthusiasts doing the same experiment.

First, [Juju](https://jujucharms.com) is a cool controller and agent based tool from Canonical to easily deploy and manage applications (called Charms) on different clouds and environments (see [how it works](https://jujucharms.com/how-it-works) for more details).

We run an academic cloud, [HUNT Cloud](ttps://www.ntnu.edu/huntgenes/hunt-cloud), where we utilize a highly available Juju deployment, in concert with [MAAS](https://maas.io), to run things like [OpenStack](https://www.openstack.org) and [Ceph](https://ceph.com). For this upgrade, we were looking forward to some of the new features in the such as cross model relations and overlay bundles.

## How to upgrade Juju (for dummies)

Upgrading a Juju environment is usually a straightforward task completed with a cup of coffee and a couple of commands. The main steps are:

1. Upgrade your Juju client (the client talking to the Juju controllers, usually on your local machine, `apt upgrade juju` or `snap refresh juju`)
2. Upgrade your Juju controller (the controller managing the agents, `juju upgrade-juju --model controller`)
3. Upgrade your Juju model (the model containing your deployed applications, `juju upgrade-juju --model <name-of-model>`)

> Check out the [official docs](https://jujucharms.com/docs/stable/models-upgrade) for a more thorough explanation.

Our task at hand was to upgrade the Juju environment from 2.1.2 to 2.3.
Step 1 was easy as can be, however the remaining steps provided a few lessons learned that might prove useful for others.

## Issue No. 1

We ran the following command to upgrade our controllers:

```shell
$ juju upgrade-juju --model controller
best version:
    2.2.9
started upgrade to 2.2.9
```

Now, if you look closely, the output above says 2.2.9, not 2.3.2 which was the latest version at the time and the one I actually wanted.
Well, the upgrade to 2.2.9 succeeded, so I continued upgrading once more by running `juju upgrade-juju --model controller` to reach 2.3.2.

This time things didn't go as smooth for the controllers and they got stuck upgrading which rendered the environment unusable.
It did however produce some rather bleak yet humorous error messages.

```shell
2018-01-30 11:15:22 WARNING juju.worker.upgradesteps worker.go:275 stopped waiting for other controllers: tomb: dying
2018-01-30 11:15:22 ERROR juju.worker.upgradesteps worker.go:379 upgrade from 2.2.9 to 2.3.2 for "machine-0" failed (giving up): tomb: dying
```

I was able to reproduce this in one of our larger staging areas and the bug got fixed  in 2.3.3 in [lp#1746265](https://bugs.launchpad.net/juju/+bug/1746265).

## Issue No. 2

So, after getting stuck with the issue above, I was encouraged to try upgrading straight to 2.3.2, skipping 2.2.9 altogether.
Juju allows you to specify the target version using the `--agent-version` flag.
The command you end up with is `juju upgrade-juju --model controller --agent-version 2.3.2`.

Sticking to good form and the rule of three, the controllers got stuck upgrading rendering the environment unusable once again.
Fortunately, it was easy to reproduce both in our staging area and on local LXD deployments so this one also got fixed in 2.3.3 in [lp#1748294](https://bugs.launchpad.net/juju/+bug/1748294).

## Issue no. 3

We gave the upgrade a new try when version 2.3.4 rolled around late in February.
Things looked good after multiple runs in staging, so I finally upgraded one of our production controllers using `juju upgrade-juju --model controller --agent-version 2.3.4`.

The upgrade process took around 15 minutes. After a lot of logspam in the controller logs and some unnerving error messages in the `juju status --model controller` output, things seemed to settle.
Almost.
We noticed charm agent failures and connection errors between the controllers and a small number of the applications in the main production Juju model containing our OpenStack and Ceph deployments.

After filing [lp#1755155](https://bugs.launchpad.net/juju/+bug/1755155), I was recommended to push on and upgrade the Juju model even though some of the charms errored out.
This approach resolved the connection errors.

The root cause was most likely [lp#1697936](https://bugs.launchpad.net/juju/+bug/1697936) which was reported last year.
It turned out 2.1 agents could fail to read from 2.2 and newer controllers.
I did eventually find a mention of the bug in the changelog for 2.2.0, however the description did not contain the error messages leaving my searches in Launchpad coming up empty.

Upgrading the model with `juju upgrade-juju --model openstack --agent-version 2.3.4` and restarting the affected agents finally did the trick and all components were running smoothly on 2.3.4.

## Afterword

To be fair to the Juju team, our production model has a decent amount of different charms and therefore a decent amount of Juju agents ([we are talking about OpenStack after all](https://checknotes.files.wordpress.com/2016/01/openstack-logical-arch-folsom.png?w=1280)).

Now you might rightfully ask, _Sandor, why on earth didn't you just upgrade the model right away as described in step 3?_
Well, I simply became a bit wary of proceeding without any easy way to rollback after running into all the previous bugs.

## Lessons learned

* Always read the changelogs. Carefully.
* Always test the upgrades. This goes both for users and the dev team.
* The upgrade UX has room for improvements when offering `apt upgrade juju`, `juju upgrade-juju --model controller`, `juju upgrade-juju --model model`, `juju upgrade-charm` and `juju upgrade-gui`.
* As things can go awry, it would be nice if `juju upgrade-juju` would tell you what it is going to do with the `--dry-run` flag as it may not pick the version you want.
* It would also be nice if there was a way to do proper dry runs or even rollback (both failed and successful) upgrades besides backing up and restoring your controllers.
* Even though the controller and the model are upgraded separately and should be able to run different versions, they can break each other.

Great thanks to Rick, Tim, John and the rest of the Juju gang from Canonical for helping out with tips, troubleshooting and fixes.
