---
layout: blog/post
title: "Zero to HA MariaDB and Docker Swarm in under 15 minutes on IBM Softlayer (or anywhere, really) <br />Part Two"
date: 2016-04-05 08:00:00
image: '/blog/assets/img/docker-machine-swarm-mariadb-love.png'
description: Part two of a two-part series on building a high-availability containerized MariaDB Galera cluster on top of a multi-master docker swarm in the cloud.
tags: docker docker-swarm docker-compose docker-machine consul mariadb softlayer
categories: docker devops
twitter_text: Zero to HA MariaDB Galera on Docker Swarm in 15 minutes part two
excerpt: Part two of a two-part series on building a high-availability containerized MariaDB Galera cluster on top of a multi-master docker swarm in the cloud.
---
[![Containers](https://farm6.staticflickr.com/5532/14403331148_e105a145e5_b.jpg)](https://flic.kr/p/nWLQxE)
{: style="align: center; margin-bottom: -30px;"}
Photo By: [Jumilla](https://www.flickr.com/photos/jumilla/)
{: style="color:gray; font-size: 80%; text-align: center; padding: 0px;"}

In [part one of this series](/zero-to-mariadb-cluster-in-docker-swarm-in-15-minutes-part-1/), we looked at getting a MariaDB cluster up and running on top of a multi-master docker swarm with Consul running on the swarm itself -- in under 15 minutes. The accompanying [github repository](https://github.com/dayreiner/docker-swarm-mariadb) includes several helper scripts geared towards getting the environment up and running rapidly in [IBM Softlayer](http://www.softlayer.com/). In this second part of the series, we'll drill-down and take a closer look at the process itself for those who use a different cloud provider, want to adapt things to suit their orchestration process, or just prefer to learn by doing things from scratch. 

# Build Goals, Revisited

- Multi-master, highly-available docker swarm cluster.
- HA Consul key-value store running on the swarm itself.
- Containerized MariaDB Galera cluster running on the swarm, natch.
- Use the btrfs storage driver (or alternately, device-mapper with LVM).
- Overlay network between MariaDB nodes for cluster communication etc.
- Percona Xtrabackup instead of rsync to reduce locking during state transfers.

# Activites

* Create the multi-master Swarm and Consul cluster
  1. [Provision the instances](#provision-instances)
  2. [Run post-provisioning scripts](#post-provisioning)
  3. [Build the Swarm nodes](#build-swarm)
  4. [Deploy Consul cluster to the Swarm](#deploy-consul)
  5. [Verify the Swarm](#verify-swarm)
* Deploy MariaDB Galera cluster on the Swarm
  1. [Create the overlay network](#create-overlay)
  2. [Bootstrap the MariaDB Galera cluster](#bootstrap-mariadb)
  3. [Verify Galera cluster membership](#verify-mariadb)
  4. [Destroy the Swarm](#destroy-swarm)

---
{: style="margin-top: -30px"}

![SwarmLove](/blog/assets/img/docker-machine-swarm-mariadb-love.png)

---
{: style="margin-bottom: 20px"}

## <a name="provision-instances"></a> Provision the instances

To provision the swarm nodes we're going to use the `generic` driver for docker-machine. To get ready for to use the generic driver, we'll need to use our provider's toolset to pre-provision the instances so we can then feed them over to machine for creation as swarm nodes. Yes, there is a [Softlayer driver](https://docs.docker.com/machine/drivers/soft-layer/) for machine, but we're not going to use it. More on that [later-on](#build-swarm), but in the mean time lets use the provisioning script to create our initial Softlayer instances.

In order to keep things self-contained, the helper script [`provision_softlayer.sh`](https://github.com/dayreiner/docker-swarm-mariadb/blob/master/scripts/provision_softlayer.sh) generates an SSH key to use with the provisioned instances, adds that key to your softlayer account and then loops through and provisions the nodes specified in your [`swarm.local`](https://github.com/dayreiner/docker-swarm-mariadb/blob/master/config/swarm.conf) file. The generated ssh key is stored in the ssh directory. To provision manually, you will need to generate an ssh key or use an existing key in your account and then pass the name of the key to `slcli` when ordering:

{% highlight bash %}
# Vars taken from swarm.local
node="sw1" # Node Name
sl_sshkey_name="mysshkey" # SSH Key to use from Control Panel
sl_domain="example.com" # Softlayer Domain
sl_cpu=1 # Number of Cores
sl_memory=1 # Amount of RAM
sl_region="tor01" # Softlayer Datacenter
sl_billing="hourly" # Hourly or Monthly Billing
sl_os_disk_size=25 # OS volume Size
sl_docker_disk_size=25 # Docker volume size
sl_public_vlan_id=123456 # Public iface VLAN ID
sl_private_vlan_id=654321 # Private iface VLAN ID

# Order the instance
slcli vs create -H ${node} -D ${sl_domain} \
    -k $(slcli sshkey list | grep ${sl_sshkey_name} | awk '{print $1}') \
    -c ${sl_cpu} -m ${sl_memory} \
    -d ${sl_region} -o CENTOS_LATEST \
    --billing ${sl_billing} --public \
    --disk ${sl_os_disk_size} \ 
    --disk ${sl_docker_disk_size} -n 100 \ 
    --vlan-public ${sl_public_vlan_id} \
    --vlan-private ${sl_private_vlan_id} \
    --tag dockerhost
{% endhighlight %}

To check when the provisioning process is completed, run:

`slcli vs detail ${node} | grep state` 

Once the API reports back a state of `RUNNING`, your node is ready.

## <a name="post-provisioning"></a> Run post-provisioning scripts

Once the intstances are provisioned, we're going to want to prep them a bit before adding them with machine. `provision_softlayer.sh` copies and executes [`sl_post_provision.sh`](https://github.com/dayreiner/docker-swarm-mariadb/blob/master/scripts/instance/sl_post_provision.sh) from the `instance` subdirectory over to the newly-provisioned instances and runs it once they are accessible.

Since we're using docker's [btrfs storage driver](https://docs.docker.com/engine/userguide/storagedriver/btrfs-driver/), `sl_post_provision.sh` runs the following actions to prep the system and build our btrfs docker volume:

* Updates the system
* Installs the necessary packages to support LVM and btrfs
* Installs the net-tools package to workaround docker-machine [issue #2481](https://github.com/docker/machine/issues/2481)
* Partitions the secondary volume on the instance
* Adds the secondary volume to LVM and formats it as a btrfs filesystem
* Mounts the btrfs filesystem at `/var/lib/docker`

> **Tip:** Depending on your environment, you may want to also run your [ansible](https://www.ansible.com/) playbooks, install additional software or further tune the system to fit your standards.

First, we'll update the system and install the necessary packages:

{% highlight bash %}
yum -y update && yum clean all && yum makecache fast
yum -y install lvm2 lvm2-libs btrfs-progs git net-tools
{% endhighlight %}

Next, we partition and format the secondary volume we'll be using for the `/var/lib/docker` directory:

{% highlight python %}
cat << EOF > /tmp/xvdc.layout
# partition table of /dev/xvde
unit: sectors

/dev/xvdc1 : start=     2048, size= 26213376, Id=8e
/dev/xvdc2 : start=        0, size=        0, Id= 0
/dev/xvdc3 : start=        0, size=        0, Id= 0
/dev/xvdc4 : start=        0, size=        0, Id= 0
EOF
{% endhighlight %}

... and convert to a logical volume under LVM:

{% highlight bash %}
sfdisk --force /dev/xvdc < /tmp/xvdc.layout
pvcreate /dev/xvdc1
vgcreate docker_vg /dev/xvdc1
lvcreate -l 100%FREE -n docker_lv1 docker_vg
{% endhighlight %}

You could stop here and just set up the device mapper driver when deploying docker to use the LVM group for the docker volume and metadata. You can find more information on configuring the device mapper driver in the official [Docker Documentation](https://docs.docker.com/engine/userguide/storagedriver/device-mapper-driver/).

Building with `sl_post_provision.sh`, we go one step further and format the LVM volume with btrfs. To configure the LVM partition as a btrfs filesystem, simply run the command 

`mkfs.btrfs /dev/docker_vg/docker_lv1`.

> **Note:** In going the btrfs route, there is also no need to manually configure a storage driver -- when docker is started it will detect that the `/var/lib/docker` directory is on a btrfs partition and automatically load the btrfs storage driver. 
> 
> There's some discussion still on whether btrfs is production-ready or not, but for noncritical environments I like the convenience of working with the btrfs [feature-set](https://en.wikipedia.org/wiki/Btrfs#Features). With Redhat (and thus CentOS) 7.1, btrfs is still considered a [technology preview](https://access.redhat.com/documentation/en-US/Red_Hat_Enterprise_Linux/7/html/Storage_Administration_Guide/ch-btrfs.html) -- so while I've not had any issues with it day-to-day, YMMV. Caveat emptor.

Once the post-provisioning process is complete, we're ready to move on to building the actual swarm nodes using `docker-machine`.

## <a name="build-swarm"></a> Build the Swarm nodes

![AUklet flock Shumagins 1986 by By D. Dibenski via Wikimedia Commons](https://upload.wikimedia.org/wikipedia/commons/5/5e/Auklet_flock_Shumagins_1986.jpg)
{: style="align: center; margin-top: -50px; margin-bottom: -20px;"}

Once the instances we'll be using have been provisioned, the next helper script, [`build_swarm.sh`](https://github.com/dayreiner/docker-swarm-mariadb/blob/master/scripts/build_swarm.sh) will create the swarm  using the `generic` machine driver and deploy consul across each of the swarm masters.

> ***So why not just use machine's built-in Softlayer driver to deploy the nodes? Why the extra step?***
>
>The primary reason we're giving it a skip is because we need to pass IP address information to docker-machine as part of the swarm creation -- right now, there's no way to do that with the standard drivers. By pre-provisioning with our provider's CLI, we can use it to grab the provisioned instance's private and public IPs to pass to machine when creating the swarm. While this is defnitely a shortcoming of the current machine drivers, provisioning with `slcli` (*or your provider's tool of choice*) offers some more flexibility with provisioning the instance with non-standard images or configurations.

For each node in the cluster, we'll run `docker-machine create` and tell it to create a master node with the `--swarm-master` switch, and tell it to replicate to the other masters via the `--swarm-opt="replication=true"` switch. We're then telling the swarm master to find the other masters by using the key-value store located at `consul://consul.service.consul`. Since we're also using the node's local IP address for our primary DNS (via `--engine-opt="dns ${node_private_ip}"`), each node will also find all of the other participating consul instances. 

{% highlight bash %}
# Vars taken from swarm.local
node=sw1
node_ssh_ip=10.1.1.1 # Public or Private IP machine will use to connect
node_private_ip=10.1.1.1
node_consul=consul.service.consul
dns_primary=8.8.8.8
dns_secondary=8.8.4.4
dns_search_domain=example.com
datacenter=tor01

docker-machine create \
    --driver generic \
    --generic-ip-address ${node_ssh_ip} \
    --generic-ssh-key ${__root}/ssh/swarm.rsa \
    --generic-ssh-user root \
    --engine-storage-driver btrfs \
    --swarm --swarm-master \
    --swarm-opt="replication=true" \
    --swarm-opt="advertise=${node_private_ip}:3376" \
    --swarm-discovery="consul://${node_consul}:8500" \
    --engine-opt="cluster-store consul://${node_consul}:8500" \
    --engine-opt="cluster-advertise=eth0:2376" \
    --engine-opt="dns ${node_private_ip}" \
    --engine-opt="dns ${dns_primary}" \
    --engine-opt="dns ${dns_secondary}" \
    --engine-opt="log-driver json-file" \
    --engine-opt="log-opt max-file=10" \
    --engine-opt="log-opt max-size=10m" \
    --engine-opt="dns-search=${dns_search_domain}" \
    --engine-label="dc=${datacenter}" \
    --engine-label="instance_type=public_cloud" \
    --tls-san ${node} \
    --tls-san ${node_private_ip} \
    --tls-san ${node_public_ip} \
    ${node}
{% endhighlight %}    

## <a name="deploy-consul"></a >Deploy Consul cluster to the Swarm

[![Consul](/blog/assets/img/consul.png)](https://www.consul.io/)
{: style="align: center; margin-top: -40px; margin-bottom: -5px;"}
In Jacob Blain Christen's article [Toward a Production-Ready Docker Swarm with Consul](https://medium.com/on-docker/toward-a-production-ready-docker-swarm-cluster-with-consul-9ecd36533bb8#.fngyb759z), he made a **very** useful discovery -- if you configure machine to point swarm at a non-existent consul address, it will still create the node succesfully and just continue to retry its connection rather than giving up.

Since we haven't deployed the actual consul containers yet, the swarm nodes can't yet form a multi-master cluster. The [`build_swarm.sh`](https://github.com/dayreiner/docker-swarm-mariadb/blob/master/scripts/build_swarm.sh) script and compose file borrow his technique to buy some time to deploy the consul containers *after* we've got the individual swarm nodes started in a semi-functional state. 

After the swarm masters have been provisioned with machine, `build_swarm.sh` will then run `docker-compose` to deploy Consul across all nodes. 

> **Note:** As this is a self-contained cluster, we're using [consul server](https://hub.docker.com/r/gliderlabs/consul-server/) images on each node -- larger swarms would be better served using the [consul agent image](https://hub.docker.com/r/gliderlabs/consul-agent/) instead on any swarm members that aren't part of the initial HA contol nodes:

{% highlight bash %}
# Vars taken from swarm.local
node=sw1
datacenter=tor01
dns_primary=8.8.8.8
dns_secondary=8.8.4.4
dns_search_domain=example.com
node_cluster_ip=10.1.1.1
othernode0_cluster_ip=10.1.1.2
othernode1_cluster_ip=10.1.1.3

docker-machine ssh ${node} "printf 'nameserver ${node_cluster_ip}\nnameserver ${dns_primary}\nnameserver ${dns_secondary}\ndomain ${dns_search_domain}\n' > /etc/resolv.conf"
docker-machine scp -r ./compose/consul/config ${node}:/tmp/consul
docker-machine ssh ${node} "mv /tmp/consul /etc"
eval $(docker-machine env ${node})
docker-compose -f ./compose/consul/consul.yml up -d consul
docker-machine ssh ${node} "systemctl restart docker"
{% endhighlight %}

This copies over the consul configuration file, changes the dockerhost's resolv.conf to use the local consul for its primary DNS and runs `docker-compose` to build the consul container using some variables we'll either pull from the `swarm.local` config or figure out automatically. After building consul, we then bounce the docker daemon so it can successfully connect to the `consul.service.consul` service in DNS. Our compose file looks like this:

{% highlight bash %}
consul:
  command: -dc ${datacenter} -server -node ${node} -client 0.0.0.0 -bootstrap-expect ${swarm_total_nodes} -advertise ${node_cluster_ip} -retry-interval 10s -recursor ${dns_primary} -recursor ${dns_secondary} -retry-join ${othernode0_cluster_ip} -retry-join ${othernode1_cluster_ip}
  container_name: consul
  net: host
  environment:
  - "GOMAXPROCS=2"
  image: gliderlabs/consul-server:latest
  ports:
  - 172.17.0.1:53:53
  - 172.17.0.1:53:53/udp
  - ${node_cluster_ip}:53:53
  - ${node_cluster_ip}:53:53/udp
  - ${node_cluster_ip}:8300-8302:8300-8302
  - ${node_cluster_ip}:8300-8302:8300-8302/udp
  - ${node_cluster_ip}:8400:8400
  - ${node_cluster_ip}:8500:8500
  restart: always
  volumes:
  - "consul-data:/data"
  - "/etc/consul/consul.json:/config/consul.json:ro"
  - "/etc/docker/ca.pem:/certs/ca.pem:ro"
  - "/etc/docker/server.pem:/certs/server.pem:ro"
  - "/etc/docker/server-key.pem:/certs/server-key.pem:ro"
  - "/var/run/docker.sock:/var/run/docker.sock"
{% endhighlight %} 

... and this consul.json

{% highlight bash %}
{
    "ca_file": "/certs/ca.pem",
    "cert_file": "/certs/server.pem",
    "key_file": "/certs/server-key.pem",

    "ports": {
        "dns": 53
    },

    "verify_incoming": true,
    "verify_outgoing": true
}
{% endhighlight %}

This exposes consul's DNS on port 53 locally on the swarm node, adds our upstream DNS as recursors for consul and tells it to join the cluster with the other consul containers living on the other swarm masters. For more information on configuring and using consul, refer to the [consul documentation](https://www.consul.io/docs/). If you're looking to perform automated service discovery, at this point you could also deploy [registrator](https://github.com/gliderlabs/registrator) to automatically populate consul with any publicly exposed services running across the swarm.

## <a name="verify-swarm"></a >Verify the Swarm

By now, `build_swarm.sh` should have exited. Within a few minutes, the restarted docker daemons should start finding the consul cluster and being the process of starting the swarm and connecting to the other swarm masters. To check on the status, load the environment variables for the swarm using `eval $(docker-machine env --swarm ${node}` and issuing a `docker info` command. If the swarm is operational, you should see output similar to below -- with each swarm member listed in the swarm node list. If the number of `Nodes` is less than what you deployed or is empty, wait a few seconds and try again:

{% highlight bash %}
Containers: 9
 Running: 9
 Paused: 0
 Stopped: 0
Images: 9
Server Version: swarm/1.1.3
Role: replica
Primary: 10.1.1.2:3376
Strategy: spread
Filters: health, port, dependency, affinity, constraint
Nodes: 3
 sw1: 10.1.1.0:2376
  └ Status: Healthy
  └ Containers: 4
  └ Reserved CPUs: 0 / 1
  └ Reserved Memory: 0 B / 1.013 GiB
  └ Labels: dc=tor01, executiondriver=native-0.2, instance_type=public_cloud, 
    kernelversion=3.10.0-327.10.1.el7.x86_64, operatingsystem=CentOS Linux 7 (Core), 
    provider=generic, storagedriver=btrfs
  └ Error: (none)
  └ UpdatedAt: 2016-03-28T14:09:03Z
 sw2: 10.1.1.1:2376
  └ Status: Healthy
  └ Containers: 4
  └ Reserved CPUs: 0 / 1
  └ Reserved Memory: 0 B / 1.013 GiB
  └ Labels: dc=tor01, executiondriver=native-0.2, instance_type=public_cloud, 
    kernelversion=3.10.0-327.10.1.el7.x86_64, operatingsystem=CentOS Linux 7 (Core), 
    provider=generic, storagedriver=btrfs
  └ Error: (none)
  └ UpdatedAt: 2016-03-28T14:09:27Z
 sw3: 10.1.1.2:2376
  └ Status: Healthy
  └ Containers: 4
  └ Reserved CPUs: 0 / 1
  └ Reserved Memory: 0 B / 1.013 GiB
  └ Labels: dc=tor01, executiondriver=native-0.2, instance_type=public_cloud,
    kernelversion=3.10.0-327.10.1.el7.x86_64, operatingsystem=CentOS Linux 7 (Core), 
    provider=generic, storagedriver=btrfs
  └ Error: (none)
  └ UpdatedAt: 2016-03-28T14:09:37Z
Plugins:
 Volume:
 Network:
Kernel Version: 3.10.0-327.10.1.el7.x86_64
Operating System: linux
Architecture: amd64
CPUs: 3
Total Memory: 3.038 GiB
Name: sw1
{% endhighlight %}

# Standing up MariaDB

[![MariaDB Galera](/blog/assets/img/galera.png)](https://mariadb.com/kb/en/mariadb/what-is-mariadb-galera-cluster/)
{: style="align: center; margin-top: -50px; margin-bottom: -20px;"}

So now we've got a multi-master swarm up and running using a high-availabilty consul cluster running directly on the swarm masters. Sweet! Next, we'll move on to getting MariaDB up and running on the swarm. To accomplish this, the [`deploy_mariadb.sh`](https://github.com/dayreiner/docker-swarm-mariadb/blob/master/scripts/deploy_mariadb.sh) script will create an overlay network on top of the swarm and then bootstrap the MariaDB cluster using `docker-compose`. 

> **Note:** Keeping with the CentOS 7 theme, we'll be using my [centos7-mariadb-10.1-galera](https://hub.docker.com/r/dayreiner/centos7-mariadb-10.1-galera/) image from Docker Hub to deploy the MariaDB Galera cluster across the swarm nodes. This image is based upon the offical MariaDB 10.1 image, except it uses CentOS insted of Ubuntu and has both Galera cluster and Percona Xtrabackup support added in. More info on customizing/configuring the image can be found at the previous link or at the [source repository](https://github.com/dayreiner/centos7-MariaDB-10.1-galera) on github.

Before we start deploying the MariaDB image though, we'll create our overlay network. The overlay network will be used for all inter-cluster communications. `deploy_mariadb.sh` will also expose port 3306 on each instance to the internal network of the dockerhost. Assuming you're running you app servers across the same swarm, you could skip exposing the port at all, and run your application traffic against the overlay network as well.

## <a name="create-overlay"></a> Create the overlay network

[![Overlay Network](/blog/assets/img/docker-overlay.png)](https://docs.docker.com/engine/userguide/networking/get-started-overlay/)
{: style="align: center; margin-top: -50px; margin-bottom: -20px;"}

Once swarm is up and running, creating the overlay network is just a matter of issuing a couple of commands and optionally picking out a subnet for your network if you need to specify it (i.e. to avoid conflicts with other local subnets). Make sure you have machine pointed at the swarm by issuing `eval $(docker-machine env --swarm ${node})`, and then create the subnet with:

{% highlight bash %}
docker network create -d overlay --subnet=172.100.100.0/24 mariadb
{% endhighlight %}

## <a name="bootstrap-mariadb"></a>Bootstrap the MariaDB Galera cluster

Now that we have our overlay network sorted, we can start getting our MariaDB Galera cluster up and running using the overlay network for cluster communications. After creating the overlay network, [`deploy_mariadb.sh`](https://github.com/dayreiner/docker-swarm-mariadb/blob/master/scripts/deploy_mariadb.sh) will dynamically generate compose files for each swarm node and bring the cluster members up. The first cluster member is brought up with a special value for the `$cluster_members` environment variable -- by setting this to `BOOTSTRAP` the first node will know to initialize a new cluster. 

> **Tip:** You can customize the `mariadb.yml` compose file to not only use a volume on the dockerhost for a persistent database, but also add any scripts you want executed on start (such as to create databases or import data) by mounting a directory containing your scripts as a container volume called `/docker-entrypoint-initdb.d`. For a full list of configurable options, please refer to the [image description](https://hub.docker.com/r/dayreiner/centos7-mariadb-10.1-galera/) on docker hub. Don't forget, since this is a cluster you'll only need to run your scripts on the first node!

As there are no nodes running, the first node will detect it is not operational and perform the bootstrap process. Running the script a second time with the cluster active will restart the first node as a regular cluster member. Passing some variables to compose, the script bootstraps the first node:

{% highlight bash %}
# Vars taken from swarm.local
mariadb_data_path=/path/to/persistent/storage
cluster_members=( array of swarm nodes taken from swarm.local )
mariadb_cluster_name=A unique name to allow for multiple clusters on the same network
mysql_root_password=secret
sst_password=alsosecret

eval $(docker-machine env ${node})
export cluster_members=BOOTSTRAP # First node - set the cluster mode to bootstrap
export node
sed "s/%%DBNODE%%/db-${node}/g" mariadb.yml > ${node}.yml
docker-compose -f ${__root}/compose/mariadb/${node}.yml up -d --no-recreate
{% endhighlight %}

...using the generated dockerfile `${node}.yml`. This places the MariaDB container on the overlay network we created earlier, exposes the necessary ports to the overlay network, and forwards port 3306 on the dockerhost to port 3306 of the MariaDB container:

{% highlight bash %}
version: '2'
services:
  db-sw1:
    image: dayreiner/centos7-mariadb-10.1-galera
    container_name: db-sw1
    hostname: db-sw1
    restart: always
    networks:
     - mariadb
    ports:
     - 172.17.0.1:3306:3306
    expose:
     - "3306"
     - "4567"
     - "4444"
    volumes:
     - ${mariadb_data_path}:/var/lib/mysql
    env_file:
     - common.env
    environment:
     # This is set by the build script
     - CLUSTER=${cluster_members}
     # These are configured in swarm.conf
     - CLUSTER_NAME=${mariadb_cluster_name}
     - MYSQL_ROOT_PASSWORD=${mysql_root_password}
     - SST_USER=sst
     - SST_PASS=${sst_password}
networks:
  mariadb:
   external:
    name: mariadb
{% endhighlight %}

Before moving on to the secondary cluster members, the script first waits for the bootstrap node to report that it is operational by looking for the log entry `Synchronized with group, ready for connections` via the `docker logs` command. The process is then repeated for the remaining nodes, except the secondary nodes are started with a list of nodes to connect to.

By deploying the DB cluster into swarm's overlay network, the built-in DNS service will automatically allow the containers to find each other by name. This allows for relatively simple scaling of the database cluster, adding new members with just a couple of commands. While not exactly true “service discovery” per-se, as long as we stick to a standard naming scheme then adding new cluster members is a trivial process that can be easily automated.

## <a name="verify-mariadb"></a>Verify Galera cluster membership

Once all of the cluster members have started, you can confirm the Galera cluster is operational by setting your environment via `eval ${docker-machine env --swarm ${node})` and running a few verification commands:

{% highlight bash %}
docker exec -ti sw1-db1 mysql -psecret "show status like 'wsrep_local_state_comment';"
{% endhighlight %}
...each cluster member should return `Synced`.

{% highlight bash %}
docker exec -ti sw1-db1 mysql -psecret "show status like 'wsrep_cluster_size';"
{% endhighlight %}
...this value should be the same as the # of nodes. So for a three-node cluster, this should return `3`.

{% highlight bash %}
docker exec -ti sw1-db1 mysql -psecret "show status like 'wsrep_local_state_uuid';"
{% endhighlight %}
...all members should report back the same UUID.

Once you have confirmed the cluster is operational, you can *optionally* re-run `./deploy_mariadb.sh` to redeploy the bootstrap node as a standard galera cluster member.

## <a name="destroy-swarm"></a>Destroying the Swarm

At this point, you may just want to clear everything and start over. To destroy the swarm (without cancelling any instances), run the `destroy_swarm.sh` script. If you provisioned your nodes in IBM Softlayer, you can use the `cancel_softlayer.sh` script to cancel and destroy the instances you provisioned using `provision_softlayer.sh`. You can also tear down and rebuild the swarm using the `rebuild_swarm.sh` script (it just calls `destroy_swarm.sh` followed by `build_swarm.sh`), which has been included for convenience when experimenting.

> **Note**: The `cancel_softlayer.sh` script (just like the ordering script) needs `expect` installed on your system to get around `slcli` not having a `-y` option to force through orders/cancellations without any input. To perform the raw API calls to order and cancel without any prompting, or for more information on using the Softlayer ordering API programatically, refer to the [documentation](https://sldn.softlayer.com/blog/phil/Simplified-CCI-Creation).

# Happy Swarming!

That's it! Hopefully this has given you a better understanding of the overall process. Dig through the [repository](https://github.com/dayreiner/docker-swarm-mariadb/), play with the [scripts](https://github.com/dayreiner/docker-swarm-mariadb/tree/master/scripts) and [compose](https://github.com/dayreiner/docker-swarm-mariadb/tree/master/compose) files, and adapt the process to your own orchestration tools, providers and processes... You'll be swarming in no time flat!

![SwarmLove](/blog/assets/img/docker-machine-swarm-mariadb-love.png)
