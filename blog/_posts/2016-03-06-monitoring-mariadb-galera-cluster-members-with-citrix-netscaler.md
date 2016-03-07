---
layout: blog/post
title: "Monitoring MariaDB Galera cluster members with Citrix Netscaler"
date: 2016-03-06 22:10:13
image: '/assets/img/'
description:
tags:
categories:
twitter_text:
---
Monitoring MariaDB Galera Cluster Members with Citrix Netscaler
=======================================
At my workplace, we run several [MariaDB Galera clusters](https://mariadb.com/kb/en/mariadb/what-is-mariadb-galera-cluster/) for various client sites and use [Citrix Netscaler](https://www.citrix.com/products/netscaler-application-delivery-controller/overview.html) Application Delivery Controllers to load-balance various services (among other features). With near-synchronous multimaster replication, Galera cluster simplifies application stacks as in many cases you can point each web server at its own corresponding database server and just worry about balancing the load between your application servers. 

Citrix netscaler also supports load-balancing across MySQL systems, for instances where you want to spread out reads across systems or want to balance reads and send writes to a specific system. 

#### But what happens when an instance dies?
If you're using a 1:1 relationship of app to db instances or if you're balancing load to the database, you'll want to make sure that not only is the database responding but also that the specific database system you're connecting to is synchronized with the cluster and not experiencing any issues. 

The standard MySQL checks in Netscaler only check if the system is alive, which isn't very useful if you want to avoid writing to a system while its either acting as a donor or has partitioned itself from the cluster. 

Instead, lets just have one check that both confirms the DB is alive and also that its a healthy member of the cluster.


# MariaDB Configuration
To perform the test, we'll be using the `show status` query to get the current cluster state for the server from the `wsrep_local_state_comment` value. The cluster status should always be `Synced`, otherwise there's a problem or something else going on. A full list of statuses can be found [here](http://galeracluster.com/documentation-webpages/nodestates.html#node-state-changes).

## Add the citrix service check user to the database
In order to run service checks against a MariaDB cluster you'll need to first add a user to the database with read-only access to show global status variables. Log on to any node in the cluster and run:
{% highlight bash %}
CREATE USER 'citrix'@'$netscaler_ip_address' IDENTIFIED BY '$some_password' ;
FLUSH PRIVILEGES ;
{% endhighlight %}
Replace `$netscaler_ip_address` with the IP address for your netscaler. If you have multiple netscalers, you'll either need one entry for each IP or you can just use the wildcard `%` to allow any IP. Since we're just using `show status` commands, no special privileges are required for the user.

# Netscaler Configuration
On the netscaler we'll need to add the database user configured above, set up our service check and then bind that check to any services that depend on a specific database instance in the cluster.

## Add the db user credentials to the netscaler
Once your service check user is active in the database, you'll also need to configure the same user on the Netscaler. Login to your netscaler and run:
{% highlight bash %}
add db user citrix -password $some_password
{% endhighlight %}
... again replacing `$some_password` with the password you set for the Mysql user. 

## Setup the service check
If you have a load-balanced DB service on the netscaler or run your MariaDB instance off the same IP address as your application server, you can create a single test and just bind that to each service. The test will automatically pick up the IP of the defined service and run the test against that. To add this type of test, run the following command on the netscaler:
{% highlight bash %}
add lb monitor check-galera-status MYSQL-ECV -userName citrix -LRTM DISABLED -deviation 2 -interval 15 -resptimeout 5 -retries 5 -failureRetries 3 -destPort 3306 -database mysql -sqlQuery "show global status like \'wsrep_local_state_comment\'" -evalRule "MYSQL.RES.ROW(0).TEXT_ELEM(1).CONTAINS(\"Synced\")"
{% endhighlight %}
Then you can bind that service to any services that run a cluster member at their corresponding IP address via:

{% highlight bash %}
bind service $service_name -monitorName check-galera-status
{% endhighlight %}
... where `$service_name` is the name of the service bound to your db or application vserver.

This test will check the database server every 15 seconds with a 5-second timeout. To avoid flapping on short state changes (such as the brief locking period when acting as a Donor with Xtrabackup SST), the check will only fail if three out of the last five attempts did not return the expected result or timed out. Feel free to change these values.

### Setup a service check for a dependent service
In cases where your load-balanced service or application is not running on the same IP address as your database, you'll need to set up a separate test for each service that is dependent on a specific database instance. 

As an example, if you have a vserver set up with three application servers (app1, app2 and app3) each pointing to one of three MariaDB cluster members (db1, db2, db3), you'll need to set up three separate tests -- one for each application server that connects to its associated database instance.

To setup a service check for such a scenario, first you would create the service check for db1:

{% highlight bash %}
add lb monitor check-galera-status-db1 MYSQL-ECV -userName citrix -LRTM DISABLED -interval 15 -resptimeout 5 -retries 5 -failureRetries 3 -destIP $db1_ip_address -destPort 3306 -database mysql -sqlQuery "show global status like \'wsrep_local_state_comment\'" -evalRule "MYSQL.RES.ROW(0).TEXT_ELEM(1).CONTAINS(\"Synced\")"
{% endhighlight %}
... where `$db1_ip_address` is the IP address for the db1 cluster member.

Then you would bind that service to the dependent application service attached to your application vserver, in this case "app1":
{% highlight bash %}
bind service app1 -monitorName check-galera-status-db1
{% endhighlight %}
Repeat the above steps for app2 and app3. Once the monitor is in place on all three systems, then if the corresponding database for each application server drops out of the cluster, the netscaler will also mark that application server as down to prevent any traffic from hitting the affected system.
