---
layout: blog/post
title: "Zero to HA MariaDB and Docker Swarm in under 15 minutes on IBM Softlayer (or anywhere, really)<br />Part One" 
date: 2016-03-30 20:00:00
image: '/blog/assets/img/docker-swarm.png'
description: Part one of a two-part series on building a high-availability containerized MariaDB Galera cluster on top of a multi-master docker swarm in the cloud.
tags: docker docker-swarm docker-compose docker-machine consul mariadb softlayer
categories: docker 
twitter_text: Zero to HA MariaDB Galera on Docker Swarm in 15 minutes part one
excerpt: It has always bothered me when reading tutorials and documentation on setting up a docker swarm with machine, the discovery service for the swarm is always sitting on its own -- usually just a single instance sitting outside the swarm itself. Instead, let's run consul on top of the swarm itself, and deploy a containerized MariaDB Galera cluster on the swarm -- and let's do it in IBM Softlayer in under 15 minutes.
---
[![Multicolored Containers](https://farm4.staticflickr.com/3121/3144199355_d478f8c316_b.jpg)](https://www.flickr.com/photos/dahlstroms/3144199355)
{: style="align: center; margin-bottom: -30px;"}
Photo By: [Håkan Dahlström](https://www.flickr.com/photos/dahlstroms/)
{: style="color:gray; font-size: 80%; text-align: center; padding: 0px;"}

*Part one of a two-part series. The second part will be posted in a couple of days...*

While I manage environments across multiple cloud platforms (and even the occasional "traditional" colo) at [my workplace](https://k2digital.com), our primary application environment is in [IBM Softlayer](http://www.softlayer.com/). I enjoy working with Softlayer -- the API is fairly robust, there's a lot of choice (even bare-metal, should that tickle your fancy) and it allows me to deploy instances in both Toronto and Montreal. Canadian data-residency is a "big deal" for many Canadian companies; which the other big players like Google, AWS<sup>[1](https://aws.amazon.com/fr/blogs/aws/in-the-works-aws-region-in-canada/)</sup> and Azure<sup>[2](https://www.microsoft.com/en-ca/web/datacentre/default.aspx)</sup> can't accomodate (*yet*).

I also deploy a fair number of docker containers, not only for dev/test but for production workloads as well. With the introduction of overlay networking and fast, repeatable deployments with `docker-machine` and `docker-compose`, docker 1.10 has really expanded the usefulness of containers in production. 

### Production Workloads, Production Problems
With production workloads come production concerns though: high-availability in particular. It has always bothered me when reading most tutorials on setting up a docker swarm with machine that the discovery service for the swarm is always sitting in some little corner on its own -- a single instance outside the swarm itself. Even the [official swarm documentation](https://docs.docker.com/swarm/install-manual/) presents this archtecture. 

Sure, you could create an HA consul cluster outside of the swarm for this but it seems rather inefficient to run three extra nodes just for a component that plays such a relatively small (but extremely critical) role within the swarm. Yes, there are ways to take the external consul instance out of the mix, but they also tend to take machine out of the mix too -- requiring you to create the swarm manually on existing docker nodes. 

![Docker](/blog/assets/img/docker-swarm.png)
{: style="align: center; margin-top: -100px; margin-bottom: -50px"}

Enter [Jacob Blain Christen](https://medium.com/@dweomer) and his article [Toward a Production-Ready Docker Swarm with Consul](https://medium.com/on-docker/toward-a-production-ready-docker-swarm-cluster-with-consul-9ecd36533bb8#.fngyb759z), which details a simple little hack to have your cake and eat it too -- deploy both swarm and the consul cluster to the swarm itself, all using `docker-machine`. Because swarm will keep retrying after the initial provisioning, you can use this trick to get the swarm up and running with a highly-available KVS, completely self-contained. Awesome! We'll definitely be taking advantage of that here. Now while I haven't had much luck getting the same consul instance to also work for discovery of services that live exclusively on the overlay network, you could also run [registrator](http://gliderlabs.com/registrator/latest/) on top of all this to advertise services exposed on the host network.

# MariaDB + Swarm = <i class="fa fa-heart"></i>

In the past, when I would set up containerized MariaDB clusters they would live on standalone docker hosts and talk directly over the backend VLAN via exposed ports. By moving the DB cluster into swarm, cluster communication can now move to the overlay network. The built-in DNS service will automatically allow the containers to find each other by name. This also allows for relatively simple scaling of the database cluster, adding new members with just a couple of commands. While not exactly true "service discovery" per-se, as long as we stick to a standard naming scheme then adding new cluster members is fairly trivial. 

On top of all this, you can use a mix of VLANs/security groups/VNets (or whatever your provider calls them) and docker overlay networks to build various n-tier architectures, only exposing the services you need to make public on the physical network. Similarly, in cases where physical separation is not a requirement, greater container density can be achieved -- taking further advantage of the ability to isolate services within overlay networks despite the hosts residing on the same subnet.

## Build Goals 

- Multi-master, highly-available docker swarm cluster on CentOS 7.
- HA Consul key-value store running on the swarm itself.
- Use the btrfs storage driver (or alternately, device-mapper with LVM).
- Containerized MariaDB Galera cluster running on the swarm, natch.
- Overlay network between MariaDB nodes for cluster communication etc.
- Percona Xtrabackup instead of rsync to reduce locking during state transfers.

![MariaDB Galera](/blog/assets/img/mariadb-galera.png)
{: style="margin-top: -30px; margin-bottom: -60px"}

---

# Putting it all Together

In the second part of this series, we'll dive in to more detail around the individual steps in the process. In the mean time, I've put up a [repository on github](https://github.com/dayreiner/docker-swarm-mariadb) with bash scripts to automate the process both as a documentation exercise and as a general proof-of-concept. In addition to the deployment helper scripts, you'll find compose files for consul and MariaDB using my [CentOS7 MariaDB Galera](https://hub.docker.com/r/dayreiner/centos7-mariadb-10.1-galera/) docker image off of [docker hub](https://hub.docker.com). 

Running through the scripts in order, you'll end up with an n-node MariaDB Galera cluster running on top of a multi-master Docker Swarm, using an HA Consul cluster for swarm discovery and btrfs container storage -- all self-contained within the *n* swarm masters. In a real production environment, you would want to consider moving the database and any services on to Swarm agent hosts, and leave the three swarm masters to the task of managing the swarm itself.

To quickly deploy your own MariaDB cluster, follow the steps outlined below. If you prefer to just read through scripts and see for yourself how things are done (or on a different platform), you can browse through the scripts directly [here](https://github.com/dayreiner/docker-swarm-mariadb/tree/master/scripts). The provisioning process was tested from a CentOS 7 host, so YMMV with other OSes -- if you run in to any issues, feel free to [report them](https://github.com/dayreiner/docker-swarm-mariadb/issues) directly on github.

### Prerequsites
In order to run the scripts, you'll need to have the the Softlayer "[slcli](https://github.com/softlayer/softlayer-python)" command-line api client tool (`pip install softlayer`) installed and configured on the system you'll be running docker-machine from (or an alternate way to get the IP addresses of your instances if provisioned elsewhere). The expect command (`yum -y install expect`) is also required to run the softlayer order script -- otherwise you can provision instances yourself manually.

## Running the scripts
To get started, first clone the repository:

{% highlight bash %}
git clone git@github.com:dayreiner/docker-swarm-mariadb.git
{% endhighlight %}

Change to the `docker-swarm-mariadb` directory and run `source source.me` to set your docker-machine storage path to the `machine` subdirectory of the repository. This will help keep everything self-contained. Next go in the `config` directory and copy the premade example `swarm.conf` file to a new file called `swarm.local`. Make any changes you need to `swarm.local`; this will override the values in the example config. The config file is used to define some variables your environment, the number of swarm nodes etc. Once that's done, you can either provision the swarm instances automatically via the softlayer provisioning script or just skip ahead to building the swarm or the MariaDB cluster and overlay network:

### Steps
1. `cd config ; cp swarm.conf swarm.local`
2. `vi swarm.local` -- and change values for your nodes and environment
3. `cd ../scripts`
4. `./provision_softlayer.sh` -- generate ssh keys, orders nodes, runs post-provisioning scripts
5. `./build_swarm.sh` -- deploys the swarm and the consul cluster
6. Wait for the swarm nodes to find the consul cluster and finish bootstrapping the swarm. Check with:
 - `eval $(docker-machine env --swarm sw1)`
 - `docker info` -- and wait for all three nodes to be listed
7. `./deploy_mariadb.sh` Bootstrap the MariaDB cluster on the swarm nodes. 
 - Check container logs to confirm all nodes have started
 - Run `docker exec -ti sw1-db1 mysql -psecret "show status like 'wsrep%';"` to confirm the cluster is happy.
9. *Optionally* run `./deploy_mariadb.sh` a second time to redeploy db1 as a standard galera cluster member.

The repo also includes scripts for tearing down the swarm, rebuilding it and cancelling the swarm instances in Softlayer when you're done. 

#### Zero to a functional Galera Cluster across three Softlayer instances:

{% highlight bash %}
    real    13m17.885s
    user    0m15.442s
    sys     0m3.577s
{% endhighlight %}

Nice! 

In part two of this series we'll go through the process manually to explain things in more detail for those who prefer to do things from scratch, use a different provider or want to adapt the process to suit their orchestration process. Happy swarming!
