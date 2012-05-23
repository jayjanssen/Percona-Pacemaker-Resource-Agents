============================================== 
How to setup Percona replication manager (PRM) 
==============================================



-----------------------
Installing the packages
-----------------------

Redhat/Centos
=============

::

   [root@host-01 ~]# yum install pacemaker corosync


On Centos 6.2, this will install Pacemaker 1.1.6 and corosync 1.4.1.

Debian/Ubuntu
=============

::

   [root@host-01 ~]# apt-get install pacemaker corosync

On Debian Wheezy, this will install Pacemaker 1.1.6 and corosync 1.4.2

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


-----------------
Configuring MySQL
-----------------

Install MySQL
=============

Install packages like you would normally do depending on the distribution you are using.

Grants
======

The following grants are needed::

   grant replication client, replication slave on *.* to repl_user@'172.30.222.%' identified by 'ola5P1ZMU';
   grant replication client, replication slave, SUPER, PROCESS, RELOAD on *.* to repl_user@'localhost' identified by 'ola5P1ZMU';
   grant select ON mysql.user to test_user@'localhost' identified by '2JcXCxKF';

Setup replication
=================

You setup the replication like you normally do, make sure replication works fine between all hosts.  With 2 hosts, a good way of checking is to setup master-master replication.  Keep in mind though that PRM will only use master-slave.  Once done, stop MySQL and make sure it doesn't start automatically after boot.

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

For the sake of simplicity 

Node attributes
---------------

The MySQL resource primitive
----------------------------

The Master slave clone set
--------------------------

The VIP primitives
------------------

Reader VIP location rules
-------------------------

Writer VIP rules
----------------

Useful Pacemaker commands
=========================

To check the cluster status
---------------------------


To view and/or edit the configuration
-------------------------------------


To change a node status
-----------------------


----------------
Testing failover
----------------

How to
======

How to add a new node
---------------------

How to repair replication
-------------------------

How to exclude a node from the master role
------------------------------------------

How to use PRM in the cloud
---------------------------

