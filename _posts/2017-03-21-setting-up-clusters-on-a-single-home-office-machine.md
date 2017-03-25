---
published: true
comments: true
header:
  image: /assets/images/patrick-lindenberg-191841.jpg
tags:
  - DSEGraph
  - Apex
  - Kudu
  - juju
  - Hadoop
categories:
  - Misc
permalink: setting-up-clusters-on-a-single-home-office-machine
title: Setting up clusters on a single host
---

{% include toc %}

# The need
With more open source distributed compute frameworks gaining momentum, one would like to setup a true cluster for various experimentation and open source contribution needs. With closer integrations being enabled for each of these frameworks, needs arise to setup multiple frameworks as a single stack. Not everyone has the luxury for access to these frameworks in a true distributed mode. 

If we have access to for one powerful computer, there are ways we can achieve setting up of a true distributed cluster using some nice frameworks in the cloud and containers space. This post describes setup of such clusters on a single machine from three different use cases perspective. 
- A situation wherein there is a dependency on the file systems support and the default file system offered is not suitable for the cluster. 
- A situation wherein the host OS is not compatible with the distributed version. 
- A situation wherein the framework needs a collection of nodes based on a peer to peer communication model and there is no mechanism to run such a cluster in a psuedo mode


# Drawbacks of current approaches
Taking Hadoop as an example, let us first consider the alternatives before we start setting a cluster up. 

- Setup and use the single node as a psuedo cluster as given [here](https://hadoop.apache.org/docs/stable/hadoop-project-dist/hadoop-common/SingleCluster.html) 
- Use AWS services
- For some hadoop distributions, there are alternatives. For example for Cloudera, this github [project](https://github.com/cloudera/clusterdock) and this [blog](https://blog.cloudera.com/blog/2016/08/multi-node-clusters-with-cloudera-quickstart-for-docker/) can be used as a good starting point.


However there are drawbacks that prevents us from using these approaches in a flexible way. Some reasons are

- The host OS is not compatible with the installers. Many a times the host OS is on a newer kernel and the standard distributions are not yet compatible with the latest OS release. 
- Psuedo mode does not help in all use cases. Example YARN applications like Apache [Apex](https://apex.apache.org/) that run on top of YARN require a true cluster for developers to monitor the progress of an application.
- Docker container based approaches do not work great as some of these containers are not flexible enough to let a volume mountpoint to be flexible. Moreoever every change to the application sitting on top of the "pseudo" distributed cluster needs to be committed as an image and re-used in subsequent startups. In a true sense , creating an image out of distributed cluster might not be a great idea. All of this needs a lot more time to manage.
- Containerized approaches do not catch up always with the release of the distribution. For example there are stacks which allow containerized versions of the stack but they are not always on the latest version
- Of course Hadoop is just an example and many other distributed software like cassandra would ideally need a cluster of machines to be used as the hosts. Since it is a peer to peer model and requires common ports to be used across all instances of the peers, the scope of setting it up in distributed mode does not arise. 
- AWS might be the solution but comes with a cost.

# Juju - The enabler

[Juju](https://www.ubuntu.com/cloud/juju) from canonical is a service modelling and deployment tool. Juju coupled with "[charms](https://jujucharms.com)" makes setup of available stacks ( referred to as juju charm) as simple as a few clicks . We are not going to use charms in this post as the stacks we are trying to deploy are not avaialble as a ready made charm. 

Juju supports well known "clouds" like AWS, Google, Bare metal and even LXD. LXD provides for an interesting context for the use cases we are interested in. The [LXD blog series](https://insights.ubuntu.com/2016/03/14/the-lxd-2-0-story-prologue) by Stéphane Graber is a very good starting point to be educated on LXD images and containers. We are going to build upon this to solve our 3 use cases to set up a personalized cluster on a single host.

# Setting up the private cloud on localhost

This post assumes the following:
- You have atleast one drive partition which would be used to serve as the storage mount point for all LXD containers
- You are running Ubuntu 16.10 or above

First we install all of the binaries required to set up our private cloud on the local host. 

~~~bash
sudo add-apt-repository -u ppa:juju/stable
sudo apt-get update
sudo apt-get install lvm2 thin-provisioning-tools lxd-client lxd juju
~~~
In the next step, we make sure the current user trying to provision containers is added to the lxd group
~~~bash
newgrp lxd
~~~
In the next step we create a fresh volume that can be used as a ext4 filesystem based storage as opposed to the default zfs based storage. For this we use lxc storage command and use the spare partition that was mentioned as a requirement earlier. In the example below we are using hosting this lvm based storage partition on /dev/sdb1. 
~~~bash
sudo lxc storage create lvmlxd lvm source=/dev/sdb1
~~~
We then add this storage to the "default" profile. The default profile is installed automatically during the install process.
~~~bash
sudo lxc profile device add default root disk path=/ pool=lvmlxd
~~~
We are now ready to set up the private cloud. 
~~~bash
sudo lxd init
~~~
In the command line wizard that ensues, the most important difference is that we choose **NOT** to create any storage - Line 1 below (as we have already configured one using lxc storage command above ) 
~~~bash
Do you want to configure a new storage pool (yes/no)? no
Would you like LXD to be available over the network (yes/no) [default=no]? no
Address to bind LXD to (not including port) [default=all]:
Port to bind LXD to [default=8443]:
Trust password for new clients:
Again:
Would you like stale cached images to be updated automatically (yes/no) [default=yes]?
Would you like to create a new network bridge (yes/no) [default=yes]?
What should the new bridge be called [default=lxdbr0]?
What IPv4 address should be used (CIDR subnet notation, “auto” or “none”) [default=auto]?
What IPv6 address should be used (CIDR subnet notation, “auto” or “none”) [default=auto]?
LXD has been successfully configured.
~~~

We then choose to set some properties to get us started on settin up the cloud
~~~bash
sudo lxc network set lxdbr0 ipv6.address none
~~~

Next we add a model. More information about a [model](https://jujucharms.com/docs/2.1/models) and [controller](https://jujucharms.com/docs/2.1/controllers) can be found on the juju documentation site. Here we are naming our model as dataplatform and we are setting up all of our "nodes" in this model.
~~~bash
juju add-model dataplatform
~~~

# Kudu - Case of non-compatible file system

Let us first consider setting up a kudu cluster. Kudu comes with a need for the underlying file system to be handling the falloc system call which supports file system hole punching i.e. a portion of the file can be marked as unwanted and the associated storage released. However the default lxd cloud setup is configured to be run on zfs file system. Since we configured our "private cloud" to be based of ext4, we are now ready to set up our kudu cluster. 

We are setting up a 3 tablet server nodes and one master node as part of our cluster. Hence we will be adding 4 "nodes" to the dataplatform model. 

~~~bash
juju add-machine -n 4
~~~

We wait for a few minutes for all of the images to be provisioned into the local cache. Use the following command to see the current status. 
~~~bash
juju status
~~~
After a while, the juju status command would show something along the following lines ()
~~~
Model         Controller        Cloud/Region         Version
dataplatform  empty-controller  localhost/localhost  2.1.1

App  Version  Status  Scale  Charm  Store  Rev  OS  Notes

Unit  Workload  Agent  Machine  Public address  Ports  Message

Machine  State    DNS            Inst id         Series  AZ
16       started  10.245.75.113  juju-1dbdca-16  xenial
17       started  10.245.75.144  juju-1dbdca-17  xenial
18       started  10.245.75.171  juju-1dbdca-18  xenial
19       started  10.245.75.175  juju-1dbdca-19  xenial
~~~
