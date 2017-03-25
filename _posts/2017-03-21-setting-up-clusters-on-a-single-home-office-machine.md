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
---

{% include toc %}

# The need
With more open source distributed compute frameworks gaining momentum, one would like to setup a true cluster for various experimentation and open source contribution needs. With closer integrations being enabled for each of these frameworks, needs arise to setup multiple frameworks as a single stack. Not everyone has the luxury for access to these frameworks in a true distributed mode. 

If we have access to for one powerful computer, there are ways we can achieve setting up of a true distributed cluster using some nice frameworks in the cloud and containers space. This post describes setup of such clusters on a single machine from three different use cases perspective. 
- A situation wherein the host OS is not compatible with the distributed version. 
- A situation wherein the framework needs a collection of nodes based on a peer to peer communication model and there is no mechanism to run such a cluster in a psuedo mode
- A situation wherein there is a dependency on the file systems support and the default file system offered is not suitable for the cluster. 

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

Juju from canoonical is a service modelling and deployment tool. Juju coupled with "charms" makes setup of avaialbe stacks in the charms repository work and run like a charm :) . We are not going to use charms in this post as the stacks we are trying to deploy are not avaialble as a charm. 

Juju supports well known "clouds" like AWS, Google, Bare metal and even LXD. LXD provides for an interesting context for the use cases we are interested in. The following blog series by Stéphane Graber is a very good starting point to be educated on LXD images and containers:  https://insights.ubuntu.com/2016/03/14/the-lxd-2-0-story-prologue/. We are going to build upon this to solve our 3 use cases to set up a personalized cluster on a single host.


