---
published: true
comments: true
layout: splash
classes:
  - dark-theme
header: null
image: /assets/images/patrick-lindenberg-191841.jpg
---

With more open source distributed compute frameworks gaining momentum, one would like to setup a true cluster for various engineering needs. Whether one would like to play around with out of the box setups or contribute to open source a strong need exists for installing and running these clusters on a local machine. 

Taking Hadoop as an example, let us first consider the alternatives. 

- Setup and use the single node as a psuedo cluster as given [here](https://hadoop.apache.org/docs/stable/hadoop-project-dist/hadoop-common/SingleCluster.html) 
- Use AWS services
- For some stacks, there exist alternatives. For example for Cloudera, this github [project](https://github.com/cloudera/clusterdock) and this [blog](https://blog.cloudera.com/blog/2016/08/multi-node-clusters-with-cloudera-quickstart-for-docker/) can be used as a good starting point.

However there are drawbacks that prevent developers from using these approaches. Some reasons are

- The host OS is not compatible with the installers. Many a times the host OS is on a newer kernel and the standard distributions are not yet compatible with the latest OS release. 
- Psuedo mode does not help in all use cases. Example YARN applications like Apache [Apex](https://apex.apache.org/) that run on top of YARN require a true cluster for developers to monitor the progress of an application.
- Docker container based approaches do not work great some of these containers are not flexible enough to let a volume mountpoint to be flexible. Moreoever every change to the application sitting on top of the "pseudo" distributed cluster needs to be committed as an image and re-used in subsequent startups. All of this needs a lot more time on the developers time schedule to manage.
- Containerized approaches do not catch up always with the release of the distribution
- Of course Hadoop is just an example and many other distributed software like cassandra ) would ideally need a cluster of machines to be used as the hosts. 
- AWS might be the solution but comes with a cost.
