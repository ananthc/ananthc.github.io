---
published: true
comments: true
header:
  image: /assets/images/patrick-lindenberg-191841.jpg
tags:
  - Apex
  - Kudu
  - juju
  - Hadoop
categories:
  - Misc
permalink: setting-up-clusters-on-a-single-home-office-machine
title: Setting up clusters on a single host
gallery:
  - url: /assets/images/cluster-on-a-single-host-juju/Apex-install.png
    image_path: /assets/images/cluster-on-a-single-host-juju/Apex-install.png
    alt: APex dashboard
    title: Apache pex running on CDH cluster
  - url: /assets/images/cluster-on-a-single-host-juju/CDH-install.png
    image_path: /assets/images/cluster-on-a-single-host-juju/CDH-install.png
    alt: CDH Admin Console
    title: CDH cluster with Spark and Impala
  - url: /assets/images/cluster-on-a-single-host-juju/kudumaster.png
    image_path: /assets/images/cluster-on-a-single-host-juju/kudumaster.png
    alt: kudu master
    title: Kudu cluster master with 3 data nodes
  - url: /assets/images/cluster-on-a-single-host-juju/dsegraph-install.png
    image_path: /assets/images/cluster-on-a-single-host-juju/dsegraph-install.png
    alt: DSE graph opscenter
    title: DSE graph running on a 3 node cluster
---

{% include toc %}

# The need
Open source distributed compute frameworks are gaining momentum and in this context one would like to setup a true cluster for various experimentation and open source contribution needs. With closer integrations being enabled for each of these frameworks, needs arise to setup multiple such frameworks on a cluster of machines that are at one's disposal. Not everyone has the luxury of access to a collection of machines to run these frameworks in a true distributed mode. 

However access to a single machine is more probable and setting up of true clusters on such a single host is quickly becoming a reality. If we have access to for one powerful computer, there are ways we can achieve setting up of a true distributed cluster using some nice frameworks that are gaining traction in the cloud and containers space. This post describes setup of such clusters on a single machine from two different use cases perspective. 
- A situation wherein there is a dependency on the file systems support and the default file system offered is not suitable for the cluster.
- A situation wherein the host OS is not compatible with the software stack version. 


# Drawbacks of current approaches
Taking Hadoop as an example, let us first consider the alternatives before we start setting a cluster up. 

- Setup and use the single node as a psuedo cluster as given [here](https://hadoop.apache.org/docs/stable/hadoop-project-dist/hadoop-common/SingleCluster.html) 
- Use AWS services
- For some hadoop distributions, there are alternatives. For Cloudera as an example, this github [project](https://github.com/cloudera/clusterdock) and this [blog](https://blog.cloudera.com/blog/2016/08/multi-node-clusters-with-cloudera-quickstart-for-docker/) can be used as a good starting point.


However there are drawbacks that prevents us from using these approaches in a flexible way. Some reasons are

- The host OS is not compatible with the installers. Many a times the host OS is on a newer kernel and the standard distributions are not yet compatible with the latest OS release. 
- Psuedo mode does not help in all use cases. Example YARN applications like Apache [Apex](https://apex.apache.org/) that run on top of YARN and Hadoop require a true cluster for developers to monitor the progress of an application.
- Docker container based approaches do not work great as some of these containers are not flexible enough to let a volume mountpoint to be flexible. Moreoever every change to the application sitting on top of the "pseudo" distributed cluster needs to be committed as an image and re-used in subsequent startups. In a true sense , creating an image out of distributed cluster might not be a great idea because of the issues involved. All of this needs a lot more time to manage.
- Containerized approaches do not catch up always with the release of the distribution. For example there are stacks which allow containerized versions of the stack but they are not always on the latest version
- Of course Hadoop is just an example and many other distributed software like Cassandra would ideally need a collection of machines to be used as the hosts. Since Cassandra is a peer to peer model and requires common ports to be used across all instances of the peers, the scope of setting it up in distributed mode on a single host does not arise. 
- AWS might be the solution but comes with a cost.

# Juju and LXD - The enablers

[Juju](https://www.ubuntu.com/cloud/juju) from canonical is a service modelling and deployment tool. Juju coupled with "[charms](https://jujucharms.com)" makes setup of available stacks ( referred to as a juju charm) as simple as a few clicks . We are not going to use charms in this post as the stacks we are trying to deploy are not avaialble as a ready made charm. 

Juju supports well known "clouds" like AWS, Google, Bare metal and even LXD. LXD provides for an interesting context for the use cases we are interested in. [Here](http://unix.stackexchange.com/questions/254956/what-is-the-difference-between-docker-lxd-and-lxc) is a good summary of how LXD is different from other containers. The [LXD blog series](https://insights.ubuntu.com/2016/03/14/the-lxd-2-0-story-prologue) by Stéphane Graber is a very good starting point to be educated on LXD images and containers. We are going to build upon LXD to solve our  use cases to set up a personalized cluster on a single host.

# Setting up the private cloud on localhost

This post assumes the following:
- You have atleast one drive partition which would be used to serve as the storage pool for all LXD containers
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
We then add this storage to the "default" profile. The default profile is installed automatically during the controller provisioning process.
~~~bash
sudo lxc profile device add default root disk path=/ pool=lvmlxd
~~~

Next we edit the default lxd config to automatically provision a higher file handle limits and unlimited memory ulimits for every container that is provisioned. This is only required where certain stacks like CDH require the OS to have higher limits by default but changing these values in the container will not take effect as the underlying host is effectively managing this at the kernel level. Hence we set this in the lxd.conf located in /etc/init/lxd.conf.dpkg-dist
~~~bash
limit nofile 65536 65536
limit memlock unlimited unlimited
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

We then choose to set some properties to get us started on setting up the cloud
~~~bash
sudo lxc network set lxdbr0 ipv6.address none
~~~

If network access is needed for all of the containers from the host or on the same lan, one needs to configure a network bridge which will be used as a parent for the default eth0 interface in all of the spawned containers. Edit /etc/network/interfaces file and add the following assuming enp60 is the name of your ethernet connection name as given by ifconfig. Note that wireless connections do not seem to work with the bridged networking connection approach. 

~~~bash
auto br0
iface br0 inet dhcp
    bridge-ifaces enp6s0
    bridge-ports enp6s0
    up ifconfig enp6s0 up

iface enp6s0 inet manual
~~~

Next edit the default lxc profile to instruct eth0 in the container to be bridged to the br0 network on the host. For this edit the default profile by issuing the command 

~~~bash
sudo lxc profile edit default
~~~

In the content , replace the 'lxdbr0' with 'br0' 

Next we initialize juju framework to use LXD as the cloud infrastructure
~~~
juju bootstrap lxd
~~~

Next we add a model. More information about a [model](https://jujucharms.com/docs/2.1/models) and [controller](https://jujucharms.com/docs/2.1/controllers) can be found on the juju documentation site. Here we are naming our model as dataplatform and we are setting up all of our "nodes" in this model.
~~~bash
juju add-model dataplatform
~~~

# Kudu - Case of non-compatible file system

Let us first consider setting up a Kudu cluster. Kudu comes with a need for the underlying file system to be handling the falloc system call with support for file system hole punching i.e. a portion of the file can be marked as unwanted and the associated storage released. However the default lxd cloud setup is configured to be run on zfs file system. Since we configured our "private cloud" to be based of ext4, we are now ready to set up our kudu cluster. 

We are setting up a 3 tablet server nodes and one master node as part of our cluster. Hence we will be adding 4 "nodes" to the dataplatform model. 

~~~bash
juju add-machine -n 4
~~~

We wait for a few minutes for all of the images to be provisioned into the local cache. Use the following command to see the current status. 
~~~bash
juju status
~~~
After a while, the juju status command would show something along the following lines.
~~~bash
Model         Controller        Cloud/Region         Version
dataplatform  empty-controller  localhost/localhost  2.1.1

App  Version  Status  Scale  Charm  Store  Rev  OS  Notes

Unit  Workload  Agent  Machine  Public address  Ports  Message

Machine  State    DNS            Inst id         Series  AZ
1       started  10.245.75.113  juju-1dbdca-1  xenial
2       started  10.245.75.144  juju-1dbdca-2 xenial
3       started  10.245.75.171  juju-1dbdca-3  xenial
4       started  10.245.75.175  juju-1dbdca-4  xenial
~~~

We can get a console access by using the "[juju ssh](https://jujucharms.com/docs/2.1/commands)" command. For example to get an ssh console on the machine 3, you would issue to get into the "node". 
~~~bash
juju ssh 3
~~~

Note that the following are automatically configured by using the juju add-machine command:
- ssh user provisiong using the host login. The host user simply needs to use "juju ssh <machine-number>" to loginto the newly spawned container
- a user named "ubuntu" with sudo permissions on the lxd container node
- administrative web console with a user login credentials automatically configured- Use "juju gui" command to get the auto configured password credentials that can be used in the administrative web console.
- IP addresses configured
- Network connectivity between all the nodes in the same model
- Host node can ping and reach to any of the ports of the containers provisioned.
- File system access is available from the host to the target container from the following path: /var/lib/lxd/storage-pools/lvmlxd/containers/<container id>/rootfs ( Note that lvmlxd represents the name of the lvm that was used to create the storage pool for lxd while initializing it) 

We are now ready to install the Kudu cluster on these 4 nodes. However there is a small optimization we would like to push in. Since we guess that there would be loads of data storage requirements for each of these nodes and the default image provisioned only has a capacity of 9.8G, we are going to add extra "disk space" to each of the 3 Kudu tablet servers. The disks are going to be mounted on the host in a directory acting as a mount point inside the running lxd container. These mount points are persisted on the images even after restarts. 

First let us make a directory on the host that will be used as a mountpoint on each of the containers. Let us say the directories on the host are as follows : 

~~~bash
mkdir -p /storage/node1/data
mkdir -p /storage/node2/data
mkdir -p /storage/node3/data
~~~

We now have to ensure that the host directory permissions are compatible with the container permissions. For this we need to obtain the userid fof root for the provisioned containers. To obtain the userid, simply list the user permissions on any of the containers mounted on /var/lib/lxd/storage-pools/lvmlxd/containers/
~~~bash
ls -alh /var/lib/lxd/storage-pools/lvmlxd/containers/
~~~
Take a note of the userid as we will be assigning the hosted directory permissions to this user. Let us say that the userid is 24001. We now set the permission of the host user directories for this user.

~~~bash
sudo chown -R 24001:24001  /storage/node1/data
sudo chown -R 24001:24001  /storage/node2/data
sudo chown -R 24001:24001  /storage/node3/data
~~~

we now mount each of these directories as a direcotry on the container. For this we use the lxc config device add command. 

~~~bash
sudo lxc config device add juju-1dbdca-1 datamountname disk path=/data source=/storage/node1/data
sudo lxc config device add juju-1dbdca-2 datamountname disk path=/data source=/storage/node2/data
sudo lxc config device add juju-1dbdca-3 datamountname disk path=/data source=/storage/node3/data
~~~

Now our 4 "machine" is ready for setup. Install Kudu as you would on a normal cluster of nodes. Assuming you are doing a package based install , the following is a set of cryptic install instructions **on each of the nodes**. Use juju ssh <machine number> to each of the containers before executing the following. 
~~~bash
Add cloudera repo as given http://archive.cloudera.com/kudu/ubuntu/xenial/amd64/kudu/cloudera.list in /etc/apt/sources.list.d/cloudera.list
sudo apt-get update
sudo apt-key adv --keyserver keyserver.ubuntu.com --recv-keys 327574EE02A818DD
sudo apt-get install kudu
sudo apt-get install kudu-master (on 4th node ) 
sudo apt-get install kudu-tserver  (on the remaining 3 nodes )
finally configure /etc/kudu/conf/<config-fie> to use the mounted /data directory
~~~

Exit from the container shells.

# Hadoop - Case of incompatible OS 

We will try to install a Cloudera managed cluster on an additional 4 nodes. We will be using 3 nodes as data nodes and the 4th node to host a lot of the "non-compute" servers.  Since the images that are provisioned by default use xenial as the base image and Cloudera does not yet support xenial, we will have an issue. We will provision trusty based images on the cluster. The trick is to specify the image name while adding the new nodes that we are using to host the hadoop nodes. 

~~~bash
juju add-machine --series="trusty" -n 4
~~~
Since this image is a different one than the previously provisioned set of images, it might take a while to provision the trusty based image.

We now mount directories on the host for the following mounts on each of the hadoop node.

- /opt ( As CDH installations stores a lot of the parcels in this location and easily consumes off the 9.8G size of the default image. )
- / data ( To host the Namenode and related files ) 
- /var/log ( To account for additional storage logs in case they accumulate a lot ) 

The above mount points are configured using the following snippets as explained earlier ( shown only for data folder mount point)

~~~bash
mkdir -p /storage/node[x]/data
sudo chown -R 24001:24001  /storage/node[x]/data
sudo lxc config device add juju-1dbdca-x datamountname disk path=/data source=/storage/node[x]/data
~~~

We now provision hadoop user that is required for the installation. We execute the following **on each of the nodes.** Use juju ssh <machine number> to login to each container. 

~~~bash
sudo useradd ananth
sudo usermod -aG sudo ananth
sudo mkdir -p /home/ananth
sudo chown -R ananth:ananth /home/ananth
~~~
Exit from the container shells. 

Next we need to ensure that this user either has the same password across all containers or can be logged in remotely using the same private key. We shall be using the private key approach to show case the aspects of file copy from host to containers or vice versa. 

We first generate the key pair that will be used to login to all of the cloudera managed containers during the install process. We generate this key pair for the user we added above.

"juju ssh" to any one of the nodes (say node 5). 
~~~bash
juju ssh 5
~~~
Inside this container, run the following
~~~bash
sudo su ananth
ssh-keygen -t rsa -b 2048
touch ~/.ssh/authorized_keys
cat ~/.ssh/id_rsa.pub >> ~/.ssh/authorized_keys 
chmod 600 ~/.ssh/authorized_keys
~~~

We now distribute the keypair across all of the containers that represent the containers for the remaining nodes of the cluster. Note that copying of the private key is not absolutely required. Copying private key around is not recommended either but should be ok as we are not copying around the host users key. We are however doing so for greater flexibility of ssh access from all of the containers equally. 

To complete this action, we copy the generated key files from the chosen node ( id 5 in our case above ) to the host machine and then push them to all of the remaining nodes. On the host machine, do the following from a directory where you would be managing the file copies. **Ensure you are not on the .ssh folder on the host** 


~~~bash
sudo lxc file pull juju-1dbdca-5/home/ananth/.ssh/id_rsa.pub .
sudo lxc file pull juju-1dbdca-5/home/ananth/.ssh/id_rsa .
sudo lxc file pull juju-1dbdca-5/home/ananth/.ssh/authorized_keys .
~~~

We now have the key pair and related files on the host. We will now push them to the remaining nodes. From the same directory on the host, issue the following commands to copy the files to the target folders in the remaining containers.

First ensure that there is a folder called ".ssh" in each of the containers under the user's ( ananth ) home directory.

~~~bash
sudo lxc file push id_rsa juju-1dbdca-6/home/ananth/.ssh/
sudo lxc file push id_rsa.pub juju-1dbdca-6/home/ananth/.ssh/
sudo lxc file push authorized_keys juju-1dbdca-6/home/ananth/.ssh/

sudo lxc file push id_rsa juju-1dbdca-7/home/ananth/.ssh/
sudo lxc file push id_rsa.pub juju-1dbdca-7/home/ananth/.ssh/
sudo lxc file push authorized_keys juju-1dbdca-7/home/ananth/.ssh/

sudo lxc file push id_rsa juju-1dbdca-8/home/ananth/.ssh/
sudo lxc file push id_rsa.pub juju-1dbdca-8/home/ananth/.ssh/
sudo lxc file push authorized_keys juju-1dbdca-8/home/ananth/.ssh/

~~~

Next we ensure that copied files are having the appropriate permissions. Perform the following **on each of the remaining containers**( other than container 5 ). First run juju ssh <machine id> to get access to the shell on that container.

~~~bash
sudo su ananth
sudo chown -R ananth:ananth /home/ananth/.ssh
sudo chmod 600 /home/ananth/.ssh/id_rsa
sudo chmod 600 /home/ananth/.ssh/authorized_keys
sudo chmod 744 /home/ananth/.ssh/id_rsa.pub
~~~

The final step is to change the ownership of the id_rsa file that is copied onto the host. This is required only because the cloudera installation wizard asks for the copy of the private key file of the ananth user while installing the cluster and fails with a very very crpytic error if the private key does not have the correct permission model. 

For this traverse to the directory on the host machine where we copied the key pair files and issue a chown & chmod to the id_rsa file so that the user uploading the key file as part of the installation process has read permissions.

Follow the instructions on the cloudera [documentation](https://www.cloudera.com/documentation/enterprise/latest/topics/cm_ig_install_path_a.html#id_vwc_vym_25)  to install the cluster on containers 5 to 8.

# Conclusion

Since the data mounts are persistent, we can also install applications like Apache Apex and datastax graph and get to same state on a reboot of the host.

An example setup of a clusters of Cloudera stack with Spark and Impala, Apache Apex running configured with this CDH, Kudu cluster with 3 data nodes and DataStax Graph ( DSE graph) on another 3 nodes look like this:

{% include gallery caption="" %}
