============================================== 
How to setup Percona replication manager (PRM) 
==============================================

Author: Yves Trudeau, Percona

June 2012

.. contents::

--------
Overview
--------

The solution we building is basically made of 4 components: Corosync, Pacemaker, the mysql resource agent and MySQL itself.  

Corosync
========

Corosync handles the communication between the nodes.  It implements a cluster protocol called Totem and communicates over UDP (default port 5405).  By default it uses multicast but version 1.4.2 also supports unicast (udpu).  Pacemaker uses Corosync as a messaging service.  Corosync is not the only communication layer that can be used with Pacemaker heartbeat is another one although its usage is going down.

Pacemaker
=========

Pacemaker is the heart of the solution, it the part managing where the logic is.  Pacemaker maitains a *cluster information base* **cib** that is a share xml databases between all the actives nodes.  The updates to the cib are send synchronously too all the nodes through Corosync.  Pacemaker has an amazingly rich set of configuration settings and features that allows very complex designs.  Without going into too much details, here are some of the features offered:
- location rules: locating resources on nodes based on some criteria
- colocating rules: colocating resources based on some criteria
- clone set: a bunch of similar resource
- master-slave clone set: a clone set with different level of members
- a resource group: a group of resources forced to be together
- ordering rules: in which order should some operation be performed
- Attributes: kind of cluster wide variables, can be permanent or transient
- Monitoring: resource can be monitored
- Notification: resource can be notified of a cluster wide change

and many more.  The Pacemaker logic works with scores, the highest score wins.  

mysql resource agent
====================

In order to manage mysql and mysql replication, Pacemaker uses a resource agent which is a bash script.  The mysql resource agent bash script supports a set of calls like start, stop, monitor, promote, etc.  That allows Pacemaker to perform the required actions.

MySQL
=====
 
The final service, the database.


-----------------------
Installing the packages
-----------------------

Redhat/Centos 6
===============

::

   [root@host-01 ~]# yum install pacemaker corosync


On Centos 6.2, this will install Pacemaker 1.1.6 and corosync 1.4.1.

Debian/Ubuntu
=============

::

   [root@host-01 ~]# apt-get install pacemaker corosync

On Debian Wheezy, this will install Pacemaker 1.1.6 and corosync 1.4.2

Redhat/Centos 5
===============

On older releases of RHEL/Centos, you have to install some external repos first:

::

   [root@host-01 ~]# wget http://download.fedoraproject.org/pub/epel/5/x86_64/epel-release-5-4.noarch.rpm
   [root@host-01 ~]# rpm -Uvh epel-release-5-4.noarch.rpm
   [root@host-01 ~]# wget -O /etc/yum.repos.d/pacemaker.repo http://clusterlabs.org/rpm/epel-5/clusterlabs.repo
   [root@host-01 ~]# yum install pacemaker corosync


On RHEL 5.8, this will install Pacemaker 1.0.12 and corosync 1.2.7.

--------------------
Configuring corosync
--------------------

Creating the cluster Authkey
============================

On **one** of the host, run the following command::

   [root@host-01 ~]# cd /etc/corosync
   [root@host-01 corosync]# corosync-keygen 


The key generator needs entropy, to speed up the key generation, I suggest you run commands in another session like ``tar cvj / | md5sum > /dev/null`` and similar.  The resulting file is ``/etc/corosync/authkey`` and its access bytes are 0400 and owner root, group root.  Copy the authkey file to the other hosts of the cluster, same location, owner and rights.

Creating the corosync.conf file
===============================

The next step is to configure the communiction layer, corosync by creating the corosync configuration file ``/etc/corosync/corosync.conf``.  Let's consider the hosts in question have eth1 on the 172.30.222.x network.  A basic corosync configuration will look like::

   compatibility: whitetank
   
   totem {
         version: 2
         secauth: on
         threads: 0
         interface {
                  ringnumber: 0
                  bindnetaddr: 172.30.222.0
                  mcastaddr: 226.94.1.1
                  mcastport: 5405
                  ttl: 1
         }
   }

   logging {
         fileline: off
         to_stderr: no
         to_logfile: yes
         to_syslog: yes
         logfile: /var/log/cluster/corosync.log
         debug: off
         timestamp: on
         logger_subsys {
                  subsys: AMF
                  debug: off
         }
   }

   amf {
         mode: disabled
   }


copy the file to both servers and start corosync with ``service corosync start``.  In order to verify corosync is working correctly, run the following command::

   [root@host-01 corosync]# corosync-objctl | grep members | grep ip
   runtime.totem.pg.mrp.srp.members.-723640660.ip=r(0) ip(172.30.222.212) 
   runtime.totem.pg.mrp.srp.members.-1042407764.ip=r(0) ip(172.30.222.193)

This shows the 2 nodes that are member of the cluster.  If you have more than 2 nodes, you should have more similar entries. If you don't have an output similar to the above, make sure iptables is not blocking udp port 5405 and inspect the content of ``/var/log/cluster/corosync.log`` for more information.

The above corosync configuration file is minimalist, it can be expanded in many ways.  For more information, ``man corosync.conf`` is your friend.

**NOTE:**  Older versions of corosync (RHEL/Centos 5) may not the members when running the *corosync-objctl* command.  You can see communication taking place with the following command (change the eth if not eth1)::

   tcpdump -i eth1 -n port 5405

And you should see output similar to the following::

   09:57:46.969162 IP 172.30.222.212.hpoms-dps-lstn > 172.30.222.193.netsupport: UDP, length 107
   09:57:46.989108 IP 172.30.222.193.hpoms-dps-lstn > 226.94.1.1.netsupport: UDP, length 119
   09:57:47.159079 IP 172.30.222.193.hpoms-dps-lstn > 172.30.222.212.netsupport: UDP, length 107

---------------------
Configuring Pacemaker
---------------------

The OS level configuration for Pacemaker is very simple, create the file ``/etc/corosync/service.d/pacemaker`` with the following content::

   service {
         name: pacemaker
         ver: 1
   }


then, you can start pacemaker with ``service pacemaker start``.  Once started, you should be able to verify the cluster status with the crm command::

   [root@host-02 corosync]# crm status
   ============
   Last updated: Thu May 24 17:06:57 2012
   Last change: Thu May 24 17:05:32 2012 via crmd on host-01
   Stack: openais
   Current DC: host-01 - partition with quorum
   Version: 1.1.6-3.el6-a02c0f19a00c1eb2527ad38f146ebc0834814558
   2 Nodes configured, 2 expected votes
   0 Resources configured.
   ============

   Online: [ host-01 host-02 ]

Here, ``host-01`` and ``host-02`` correspond to the ``uname -n`` values.

-----------------
Configuring MySQL
-----------------

Installation of MySQL
=====================

Install packages like you would normally do depending on the distribution you are using.  The minimal requirements for my.cnf are a unique ``server_id`` for replication, ``log-bin`` to activate the binary log and **not** ``log-slave-updates`` since this screw up the logic.  Also, make sure pid-file and socket correspond to what will be defined below for the configuration of the mysql primitive in Pacemaker.  In our example, on Centos 6 servers::

   [root@host-01 ~]# cat /etc/my.cnf 
   [client]
   socket=/var/run/mysqld/mysqld.sock
   [mysqld]
   datadir=/var/lib/mysql
   socket=/var/run/mysqld/mysqld.sock
   user=mysql
   # Disabling symbolic-links is recommended to prevent assorted security risks
   symbolic-links=0
   log-bin
   server-id=1
   pid-file=/var/lib/mysql/mysqld.pid


Start Mysql manually with ``service mysql start`` or the equivalent.

Required Grants
===============

The following grants are needed::

   grant replication client, replication slave on *.* to repl_user@'172.30.222.%' identified by 'ola5P1ZMU';
   grant replication client, replication slave, SUPER, PROCESS, RELOAD on *.* to repl_user@'localhost' identified by 'ola5P1ZMU';
   grant select ON mysql.user to test_user@'localhost' identified by '2JcXCxKF';

Setup replication
=================

You setup the replication like you normally do, make sure replication works fine between all hosts.  With 2 hosts, a good way of checking is to setup master-master replication.  Keep in mind though that PRM will only use master-slave.  Once done, stop MySQL and make sure it doesn't start automatically after boot.  In the future, Pacemaker will be managing MySQL

-----------------------
Pacemaker configuration
-----------------------

Downloading the latest MySQL RA
===============================

The PRM solution requires a specific Pacemaker MySQL resource agent.  The new resource agent is available in version 3.9.3 of the resource-agents package.  In the Centos version used for this documentation, the version of this package is::

   [root@host-01 corosync]# rpm -qa | grep resour
   resource-agents-3.9.2-7.el6.i686

which will not do.  Since it is very recent, we can just download the latest agent from github like here::

   [root@host-01 corosync]# cd /usr/lib/ocf/resource.d/
   [root@host-01 resource.d]# mkdir percona
   [root@host-01 resource.d]# cd percona/
   [root@host-01 percona]# wget -q https://github.com/y-trudeau/resource-agents-prm/raw/master/heartbeat/mysql
   [root@host-01 percona]# chmod u+x mysql

The procedure must be repeated on all hosts.  We have created a "percona" directory to make sure there would be no conflict with the default MySQL resource agent if the resource-agents package is updated.

Configuring Pacemaker
=====================

Cluster attributes
------------------

For the sake of simplicity we start by a 2 nodes cluster.  The problem with a 2 nodes cluster is the loss of quorum as soon as one of the hosts is down.  In order to have a functional 2 nodes we must set the *no-quorum-policy* to ignore like this::

   crm_attribute --attr-name no-quorum-policy --attr-value ignore

This can be revisited for larger clusters.  Also, since for this example we are not configuring any stonith devices, we have to disable stonith with::

   crm_attribute --attr-name stonith-enabled --attr-value false

IP configuration for replication
--------------------------------

The PRM solution needs to know which IP it should use to connect to a master when configuring replication, basically, for the *master_host* parameter of the ``change master to`` command.  There's 2 ways of configuring the IPs.  

The default way is to make sure the host names resolves correctly on all the members of the cluster.  Collect the hostnames with ``uname -n`` and verify those names resolve to the IPs you want to from all hosts using replication.  If possible, avoid DNS and use /etc/hosts since DNS adds a big single point of failure.

The other way uses a node attribute.  For example, if the MySQL resource primitive name (next section) is ``p_mysql`` then you can add ``p_mysql_mysql_master_IP`` (``_mysql_master_IP`` concatenated to the resource name) to each node with the IP you want to use. Here's an example::

   node host-01 \
         attributes p_mysql_mysql_master_IP="172.30.222.193"
   node host-02 \
         attributes p_mysql_mysql_master_IP="172.30.222.212"
   
Which means the IP 172.30.222.193 will be use for the ``change master to`` command when host-01 is the master and same for 172.30.222.212, which will be used when host-02 is the master.  These IPs correspond to the private network (eth1) of those hosts.  The best way to modify the Pacemaker configuration is with the command ``crm configure edit`` which loads the configuration in vi.  Once done editing, save the file ":wq" and the new configuration will be loaded by Pacemaker.

**NOTE:** Older versions of corosync (RHEL/Centos 5) may trigger an error like the following::

   /var/run/crm/cib-invalid.vlD2Dq:14: element instance_attributes: Relax-NG validity error : Type ID doesn't allow value 'host-01-instance_attributes'
   /var/run/crm/cib-invalid.vlD2Dq:14: element instance_attributes: Relax-NG validity error : Element instance_attributes failed to validate content
   ...

In this case, ``vi`` many not work for attribute editing so you can use a command like the following to set the IP (or other attributes)::

   crm_attribute -l forever -G --node host-01 --name p_mysql_mysql_master_IP -v "172.30.222.193"

The MySQL resource primitive
----------------------------

We are now ready to start giving work to Pacemaker the first thing we will do is configure the mysql primitive which defines how Pacemaker will call the mysql resource agent.  The resource has many parameter, let's first review them, the defautls presented are the ones for Linux.

=======================  ========================================================================================================
Parameter                Description
=======================  ========================================================================================================
binary                   Location of the MySQL server binary. Typically, this will point to the mysqld or the mysqld_safe file.  
                         The recommended value is the the path of the the mysqld binary, be aware it may not be the defautl.
                         *default: /usr/bin/safe_mysqld*

client_binary            Location of the MySQL client binary.  *default: mysql*

config                   Location of the mysql configuation file. *default: /etc/my.cnf*

datadir                  Directory containing the MySQL database *default: /var/lib/mysql*

user                     Unix user under which will run the MySQL daemon *default: mysql*

group                    Unix group under which will run the MySQL daemon *default: mysql*

log                      The logfile to be used for mysqld. *default: /var/log/mysqld.log*

pid                      The location of the pid file for mysqld process. *default: /var/run/mysql/mysqld.pid*

socket                   The MySQL Unix socket file. *default: /var/lib/mysql/mysql.sock*

test_table               The table used to test mysql with a ``select count(*)``. *default: mysql.user*

test_user                The MySQL user performing the test on the test table.  Must have ``grant select`` on the test table.
                         *default: root*

test_passwd              Password of the test user. *default: no set*

enable_creation          Runs ``mysql_install_db`` if the datadir is not configured. *default: 0 (boolean 0 or 1)*  

additional_parameters    Additional MySQL parameters passed (example ``--skip-grant-tables``). *default: no set*

replication_user         The MySQL user to use in the ``change master to master_user`` command.  The user must have 
                         REPLICATION SLAVE and REPLICATION CLIENT from the other hosts and SUPER, REPLICATION SLAVE,
                         REPLICATION CLIENT, and PROCESS from localhost.  *default: no set*

replication_passwd       The password of the replication_user. *default: no set*

replication_port         TCP Port to use for MySQL replication. *default: 3306*

max_slave_lag            The maximum number of seconds a replication slave is allowed to lag behind its master. 
                         Do not set this to zero. What the cluster manager does in case a slave exceeds this maximum lag 
                         is determined by the evict_outdated_slaves parameter.  If evict_outdated_slaves is true, slave is 
                         stopped and if false, only a transcient attribute (see reader_attribute) is set to 0.

evict_outdated_slaves    This parameter instructs the resource agent how to react if the slave is lagging behind by more
                         than max_slave_lag.  When set to true, outdated slaves are stopped.  *default: false*

reader_attribute         This parameter sets the name of the transient attribute that can be used to adjust the behavior
                         of the cluster given the state of the slave.  Each slaves updates this attributor at each
                         monitor call and sets it to 1 is sane and 0 if not sane.  Sane is defined as lagging by less than
                         max_slave_lag and slave threads are running.  *default: readable*

=======================  ========================================================================================================                      

So here's a typical primitive declaration::

   primitive p_mysql ocf:percona:mysql \
         params config="/etc/my.cnf" pid="/var/lib/mysql/mysqld.pid" socket="/var/run/mysqld/mysqld.sock" replication_user="repl_user" \
                replication_passwd="ola5P1ZMU" max_slave_lag="60" evict_outdated_slaves="false" binary="/usr/libexec/mysqld" \
                test_user="test_user" test_passwd="2JcXCxKF" \
         op monitor interval="5s" role="Master" OCF_CHECK_LEVEL="1" \
         op monitor interval="2s" role="Slave" OCF_CHECK_LEVEL="1" \
         op start interval="0" timeout="60s" \
         op stop interval="0" timeout="60s" 

An easy way to load the above fragment is to use the ``crm configure edit`` command.  You will notice that we also define two monitor operations, one for the role Master and one for role slave with different intervals.  It is important to have different intervals, for Pacemaker internal reasons. Also, I defined the timeout for start and stop to 60s, make sure you have configured innodb_log_file_size in a way that mysql can stop in less than 60s with the maximum allowed number of dirty pages and that it can start in less than 60s while having to perform Innodb recovery.  Since the snippet refers to role Master and Slave, you need to also include the master slave clone set (below).

The Master slave clone set
--------------------------

Next we need to tell Pacemaker to start a set of similar resource (the p_mysql type primitive) and consider the primitives in the set as having 2 states, Master and slave.  This type of declaration uses the ``ms`` type (for master-slave).  The configuration snippet for the ``ms`` is::

   ms ms_MySQL p_mysql \
        meta master-max="1" master-node-max="1" clone-max="2" clone-node-max="1" notify="true" globally-unique="false" target-role="Master" is-managed="true"

Here, the importants elements are clone-max and notify.  ``clone-max`` is the number of databases node involded in the ``ms`` set.  Since we are consider a two nodes cluster, it is set to 2.  If we ever add a node, we will need to increase ``clone-max`` to 3.  The solution works with notification, so it is mandatory to enable notifications with ``notify`` set to true.

The VIP primitives
------------------

Let's assume we want to have a writer virtual IP (VIP), 172.30.222.100 and two reader virtual IPs, 172.30.222.101 and 172.30.222.102.  The first thing we need to do is to add the primitives to the cluster configuration.  Those primitives will look like::

   primitive reader_vip_1 ocf:heartbeat:IPaddr2 \
         params ip="172.30.222.101" nic="eth1" \
         op monitor interval="10s"
   primitive reader_vip_2 ocf:heartbeat:IPaddr2 \
         params ip="172.30.222.102" nic="eth1" \
         op monitor interval="10s"
   primitive writer_vip ocf:heartbeat:IPaddr2 \
         params ip="172.30.222.100" nic="eth1" \
         op monitor interval="10s"

After adding these primitives to the cluster configuration with ``crm configure edit``, the VIPs will be distributed in a round-robin fashion, not exactly ideal.  This is why we need to add rules to control on which hosts they'll be on.

Reader VIP location rules
-------------------------

One of the new element introduced with this solution is the addition of a transient attribute to control if a host is suitable to host a reader VIP.  The replication master are always suitable but the slave suitability is determine by the monitor operation which set the transient attribute to 1 is ok and to 0 is not.  In the MySQL primitive above, we have not set the *reader_attribute* parameter so we are using the default value "readable" for the transient attribute.  The use of the transient attribute is through a location rule which will but a score on -infinity for the VIPs to be located on unsuitable hosts.  The location rules for the reader VIPs are the following::

   location loc-no-reader-vip-1 reader_vip_1 \
         rule $id="rule-no-reader-vip-1" -inf: readable eq 0
   location loc-No-reader-vip-2 reader_vip_2 \
         rule $id="rule-no-reader-vip-2" -inf: readable eq 0

Again, use ``crm configure edit`` to add the these rules.

Writer VIP rules
----------------

The writer VIP is simpler, it is bound to the master.  This is achieved with a colocation rule and an order like below::  

   colocation writer_vip_on_master inf: writer_vip ms_MySQL:Master 
   order ms_MySQL_promote_before_vip inf: ms_MySQL:promote writer_vip:start

All together
------------

Here's all the snippets grouped together::

   [root@host-01 ~]# crm configure show
   node host-01 \
         attributes p_mysql_mysql_master_IP="172.30.222.193"
   node host-02 \
         attributes p_mysql_mysql_master_IP="172.30.222.212"
   primitive p_mysql ocf:percona:mysql \
         params config="/etc/my.cnf" pid="/var/lib/mysql/mysqld.pid" socket="/var/run/mysqld/mysqld.sock" replication_user="repl_user" replication_passwd="ola5P1ZMU" max_slave_lag="60" evict_outdated_slaves="false" binary="/usr/libexec/mysqld" test_user="test_user" test_passwd="2JcXCxKF" \                                                                                           
         op monitor interval="5s" role="Master" OCF_CHECK_LEVEL="1" \
         op monitor interval="2s" role="Slave" OCF_CHECK_LEVEL="1" \
         op start interval="0" timeout="60s" \
         op stop interval="0" timeout="60s"
   primitive reader_vip_1 ocf:heartbeat:IPaddr2 \
         params ip="172.30.222.101" nic="eth1" \
         op monitor interval="10s"
   primitive reader_vip_2 ocf:heartbeat:IPaddr2 \
         params ip="172.30.222.102" nic="eth1" \
         op monitor interval="10s"
   primitive writer_vip ocf:heartbeat:IPaddr2 \
         params ip="172.30.222.100" nic="eth1" \
         op monitor interval="10s"
   ms ms_MySQL p_mysql \
         meta master-max="1" master-node-max="1" clone-max="2" clone-node-max="1" notify="true" globally-unique="false" target-role="Master" is-managed="true"
   location loc-No-reader-vip-2 reader_vip_2 \
         rule $id="rule-no-reader-vip-2" -inf: readable eq 0
   location loc-no-reader-vip-1 reader_vip_1 \
         rule $id="rule-no-reader-vip-1" -inf: readable eq 0
   colocation writer_vip_on_master inf: writer_vip ms_MySQL:Master
   order ms_MySQL_promote_before_vip inf: ms_MySQL:promote writer_vip:start
   property $id="cib-bootstrap-options" \
         dc-version="1.1.6-3.el6-a02c0f19a00c1eb2527ad38f146ebc0834814558" \
         cluster-infrastructure="openais" \
         expected-quorum-votes="2" \
         no-quorum-policy="ignore" \
         stonith-enabled="false" \
         last-lrm-refresh="1338928815"
   property $id="mysql_replication" \
         p_mysql_REPL_INFO="172.30.222.193|mysqld-bin.000002|106"


You'll notice toward the end, the ``p_mysql_REPL_INFO`` attribute (the value may differ) that correspond to the master status when it has been promoted to master.  
 

Useful Pacemaker commands
=========================

To check the cluster status
---------------------------

Two tools can be used to query the cluster status, ``crm_mon`` and ``crm status``.  They produce the same output but ``crm_mon`` is more like top, it stays on screen and refreshes at every changes.  ``crm status`` is a one time status dump.  The output is the following::

   [root@host-01 ~]# crm status
   ============
   Last updated: Tue Jun  5 17:09:01 2012
   Last change: Tue Jun  5 16:43:08 2012 via cibadmin on host-01
   Stack: openais
   Current DC: host-01 - partition with quorum
   Version: 1.1.6-3.el6-a02c0f19a00c1eb2527ad38f146ebc0834814558
   2 Nodes configured, 2 expected votes
   5 Resources configured.
   ============

   Online: [ host-01 host-02 ]

   Master/Slave Set: ms_MySQL [p_mysql]
      Masters: [ host-01 ]
      Slaves: [ host-02 ]
   reader_vip_1   (ocf::heartbeat:IPaddr2):       Started host-01
   reader_vip_2   (ocf::heartbeat:IPaddr2):       Started host-02
   writer_vip     (ocf::heartbeat:IPaddr2):       Started host-01

To view and/or edit the configuration
-------------------------------------

To view the current configuration use ``crm configure show`` and to edit, use ``crm configure edit``.  The later command starts the vi editor on the current configuration.  If you want to use another editor, set the EDITOR session variable. 

To change a node status
-----------------------



----------------
Testing failover
----------------

How to
======

How to add a new node
---------------------

Adding a new node to the corosync and pacemaker cluster will follow the steps listed above that describe installing the packages, configuring corosync, and starting the corosync and pacemaker services.  It should **not** be necessary to re-add the crm configuration again to the pacemaker cluster.  Once the pacemaker crm config is added, the cluster is responsible for maintaining it on all members.  

Before you add the new node, however, you *should* tell pacemaker that you don't want it to come online immediately by adding the ``standby="on"`` attribute.  You can do this by adding a line to the crm config similar to the following::

	node host-09 \
		attributes ...other attributes here... standby="on"

Once the new node has joined the cluster, you need to let the ``ms`` resource know that it can have another clone (slave).  You can achieve this by increasing the ``clone-max`` attribute by one.

::

   ms ms_MySQL p_mysql \
        meta master-max="1" master-node-max="1" clone-max="3" clone-node-max="1" notify="true" globally-unique="false" target-role="Master" is-managed="true"

Note that the easiest way to make this configuration change is with ``crm configure edit``, which allows you to edit the existing configuration in the EDITOR of your choice.  You may also want to put the pacemaker cluster into maintenance-mode first:

::

	crm(live)configure# property maintenance-mode=on
	crm(live)configure# commit

If the new node is added successfully to the existing corosync ring and pacemaker cluster, then it should appear in the ``crm status`` and be in the ``standby`` status.  Taking the cluster out of ``maintenance-mode`` should be safe at this point, but be sure to leave your new node in ``standby``.

Once the cluster is out of maintenance and the new node shows up in the configuration, you need to manually clone the new slave and set it up to replicate from whichever node is the active master.  This document will not cover the basics of cloning a slave.  Note that you will have to manually start mysql on your new node (be careful to do this exactly as pacemaker does it on the other nodes) once you have a full copy of the mysql data and before you execute your ``CHANGE MASTER ...; SLAVE START;``

Verify that the new node is working, replication is consistent, and allow it to catch up using standard methods.  Once it is caught up:

#. Shutdown the manually started mysql instance.  ``mysqladmin shutdown`` may be helpful here.
#. Bring the node 'online' in pacemaker.  ``crm node online new_node_name``

The trick here is that PRM will not re-issue a CHANGE MASTER if it detects that the given mysql instance was already replicating from the current master node.  Once this node is online, then it should behave as other slave nodes and failover (and possibly be promoted to the master) accordingly.


How to repair replication
-------------------------

Repairing replication is an advanced mysql replication topic, which won't be covered in detail here.  However, it should be noted that there are two basic methods to repairing replication:

#. Inline repair (i.e., tools like `pt-table-sync`)
#. Repair by slave reclone (i.e., throw the slave's data away and re-clone it from the master or another slave )


Inline repairs should not require any PRM intervention.  As far as PRM is concerned, it is all normal replication traffic.

Reclone repairs will end up following similar steps to the ``How to add a new node`` steps above.  See above for details, but the basic steps are:

#. Put the offending slave into standby
#. Effect whatever repairs/data copying necessary
#. Bring the slave up manually, configure replication, and wait for it to catch up
#. Shutdown mysql on the slave
#. Bring the slave online in Pacemaker


How to exclude a node from the master role
------------------------------------------

Pacemaker offers a very powerful configuration language to do exactly this, and many variations are possible.   The simplest way is to simply assign a negative priority to the ms Master role and the node you want to exclude::

	location avoid_being_the_master ms_MySQL:Master -1000: my_node

This should downgrade the possiblity of ``my_node`` being the master unless there simply are no other candidates.  To prevent ``my_node`` from becoming the master ever, simply take it further::

	location never_be_the_master ms_MySQL:Master -inf: my_node

**THE ABOVE DOESN'T WORK, PENDING A RESPONSE FROM THE PACEMAKER LIST**

How to verify why a reader VIP is not on a slaves
-------------------------------------------------

How to clean up error in pacemaker
----------------------------------

Enabling trace in the resource agent
------------------------------------




Advanced topics
===============

VIPless cluster (cloud)
-----------------------

Non-multicast cluster (cloud)
-----------------------------

Stonith devices
---------------

Preventing a collapse of the slaves
-----------------------------------

Performing rolling restarts for config changes
----------------------------------------------

Because failover is automated on the PRM cluster, performing rolling configuration changes that require mysql restart (i.e., not dynamic variables) is fairly straightforward:

#. Set the node to standby
#. Make configuration changes
#. Set the node to online
#. Go to the next node

Backups with PRM
----------------

There are a few basic ways to take a mysql backup, so depending on your method it will affect what steps you need to take in pacemaker (if any).

If MySQL can continue running and the load of the backup is not a problem for continuing service on the slave, then you don't need to do anything.  Simply take your backup and allow normal service to continue.

If you need to shift production traffic away from the node (i.e., a reader vip), then simply move the resource to some other node::

	crm move slave_vip_running_on_backup_node not_the_backup_node

Perform your backup here (note replication will remain running, but tools like mysqldump should not have a problem with this because it either locks the tables or wraps its backup in a transaction).  Then, to allow pacemaker to resume management of that vip::

	crm unmove the_slave_vip_you_moved


If you need to fully shutdown mysql to take your backup, it's best to simply standby the node::

	crm node standby backup_node

*further topics*:

+ Determining good backup candidate (i.e., not the master)
+ Prohibiting the selected backup node from being eligible for the master during the backup.