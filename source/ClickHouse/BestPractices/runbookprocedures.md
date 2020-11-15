---
title: Runbook Procedures
layout: default
published: true
order: 5
---
## Raising a Support Ticket

Altinity accepts support cases via the Altinity Zendesk Portal, email, or Slack.  

To log a ticket in the support portal: 



1. Login to [https://altinity.zendesk.com](https://altinity.zendesk.com) using your email address. 
2. Press the “Add +” button to open a case. 
3. Enter case topic and details.  Please include relevant information such as:
    1. ClickHouse version
    2. Error messages
    3. How to recreate the problem if you know

You can log a ticket by sending the above information to [mailto:support@altinity.com](mailto:support@altinity.com) using your registered email address or to the shared Slack channel if available. 


## Scheduled Tasks

The following tasks include standard planned procedures for ClickHouse installations.


### Installing ClickHouse

ClickHouse is distributed in .rpm, .deb, and .tgz format.  See the [ClickHouse system documentation](https://clickhouse.tech/docs/en/getting-started/install/) for installation procedures on each. 

Best practice is to install an approved Altinity Stable release.  These releases have been tested for stability and are supported for bug fixes.  See the [ClickHouse Altinity Stable Release](https://www.altinity.com/releases) page for more information. 

Here are sample commands for Ubuntu to install a recent Altinity Stable release.


```
version=20.3.15.133
sudo apt install clickhouse-client=$version \
  clickhouse-server=$version \
  clickhouse-common-static=$version
sudo systemctl start clickhouse-server
```



### Upgrading ClickHouse **(Restart Required)**

Upgrade of ClickHouse production systems is generally simple but requires planning.  Before upgrading, follow the steps below.



1. Check if version you plan to use is stable
    1. Open a support case to check for upgrade issues.
    2. Check changelog to ensure that no configuration changes are needed.
2. Perform upgrade in staging and test applications.
3. Test downgrade in staging, in case you run into problems. 
4. Start with canary updates (one replica, after that one shard).
5. Check that everything works properly after each node upgrade. 
    3. No errors in logs.
    4. Applications healthy
6. Update rest of cluster

To upgrade a single server, follow the steps shown below. 



1. Upgrade ClickHouse packages on Linux distribution to new version.
2. Restart ClickHouse

Commands to upgrade an Ubuntu server on Ubuntu are shown below. 

 


```
# Set new version.
version=20.3.15.133
sudo apt install clickhouse-client=$version \
  clickhouse-server=$version \
  clickhouse-common-static=$version
sudo systemctl restart clickhouse-server
```



### Downgrading ClickHouse **(Restart Required)**

To downgrade a ClickHouse server, upgrade packages using the lower version.  Example for Ubuntu:


```
# Set old version.
version=19.16.19.85
sudo apt install clickhouse-client=$version \
  clickhouse-server=$version \
  clickhouse-common-static=$version
sudo systemctl restart clickhouse-server
```



### Creating a ClickHouse cluster

ClickHouse clusters are linked by shared configuration values that define layouts of shards and replicas.  Follow the steps shown below to configure ClickHouse nodes into a cluster. 



1. Configure Zookeeper and add locations in zookeeper.xml file.  (See procedures for this below.)
2. Configure proper hostnames on ClickHouse nodes
    1. Run ‘hostname --fqdn’ to ensure host name is set.
    2. Ensure nodes can connect each other using those hostnames (run ping or ssh). 
3. On each server put a file macros.xml in /etc/clickhouse-server/config.d/ folder.   These are used to assign per-server settings in tables.

    ```
    <yandex>
        <!--
             Macros are defined per server,
             and they can be used in DDL, to make the DB schema
             cluster/server neutral
         -->
        <macros>
            <cluster>prod_cluster</cluster>
            <shard>01</shard>
            <replica>clickhouse-sh1r1</replica> 
        </macros>
    </yandex>
    ```


1. On each server fill out file cluster.xml in /etc/clickhouse-server/config.d/ folder with cluster, shard, and replica definitions.  This is the main cluster for distributed tables.  \
 \
Take into account that credentials to access to remote servers (it is _user_ and _password_ fields in _replica_ node) intentionally missed to be used passwordless default-user (see details below). 

    ```
    <yandex>
        <remote_servers>
            <prod_cluster> <!-- you need to give a name for cluster -->
                <shard>
                    <internal_replication>true</internal_replication>
                    <replica>
                        <host>clickhouse-sh1r1</host>
                        <port>9000</port>
                    </replica>
                    <replica>
                        <host>clickhouse-sh1r2</host>
                        <port>9000</port>
                    </replica>
                </shard>
                <shard>
                    <internal_replication>true</internal_replication>
                    <replica>
                        <host>clickhouse-sh2r1</host>
                        <port>9000</port>
                    </replica>
                    <replica>
                        <host>clickhouse-sh2r2</host>
                        <port>9000</port>
                    </replica>
                </shard>
                <shard>
                    <internal_replication>true</internal_replication>
                    <replica>
                        <host>clickhouse-sh3r1</host>
                        <port>9000</port>
                    </replica>
                    <replica>
                        <host>clickhouse-sh3r2</host>
                        <port>9000</port>
                    </replica>
                </shard>
            </prod_cluster>
        </remote_servers>
    </yandex>
    ```


2. Add additional cluster definitions to cluster.xml
    1. all_shards -- Puts every node in a separate shard. This cluster is useful for grouping system tables and other tables that exist as standalones on each node. You can query all of them by putting a distributed table over them. 
    2. all_replicated -- Puts all nodes in a single shard as replicas.  This is optional since you don’t normally put distributed tables over tables replicated to every shard. Instead you can query them locally on each node. 
3. Test the cluster to ensure definitions are correct.  Create the following table definitions.  \
 \
Do not name a distributed table the same as the database, it leads to error 'Distributed table test looks at itself''. \


    ```
    CREATE TABLE test_local ON CLUSTER '{cluster}' (
  id UInt8, ts DateTime)
Engine=ReplicatedMergeTree(
  '/clickhouse/tables/{database}/{shard}/{table}', '{replica}') 
ORDER BY (id)
;
    CREATE TABLE test ON CLUSTER '{cluster}' 
AS test_local
Engine=Distributed('{cluster}', default, test_local);
    ```


4. Insert data into the table(s) and ensure it replicates as expected. 


### Adding a new shard to an existing ClickHouse cluster



1. Install ClickHouse on each node that will belong to the shard. 
2. Edit clusters.xml and add nodes as replicas in a new &lt;shard> tag.  
3. Add nodes to all_shards and all_replicas clusters. 
4. Propagate clusters.xml to all hosts. 
5. Manually create schema on each replica in the shard.  (Schema does not replicate automatically.)
    1. You can also issue a variation of CREATE TABLE IF NOT EXISTS statements like the following to add schema on any node that is missing it. 

        ```
        CREATE TABLE IF NOT EXISTS default.foo1 ON CLUSTER `{cluster}`
        (
            `id` UInt32
        )
        ENGINE = TinyLog

        ```



### Adding a replica to an existing ClickHouse cluster



1. Install ClickHouse on a new replica node.
2. Edit clusters.xml and add node as replica in &lt;shard> to which it will belong.  
3. Add nodes to all_shards and all_replicas clusters. 
4. Propagate new clusters.xml to all hosts.
5. Manually create schema on each new replica or use the CREATE TABLE IF NOT EXISTS command show above. 


### Adding a user to ClickHouse

User and profile settings are documented [here](https://clickhouse.tech/docs/en/operations/settings/settings-users/) in the ClickHouse documentation. To add a new user, follow steps shown below. 



1. Place the new user in a file with the same name in the users.d directory, e.g., /etc/clickhouse-server/users.d/myuser.xml. 
2. Add user settings. Best practices:
    1. All accounts other than default should have passwords. 
    2. Use sha256 hash instead of plaintext passwords.
    3. Restrict network accessibility for any users with null passwords.  The **default **user is special, since it is used by default for query between servers.  It is common to leave the password null, which means it must be secured in a different way. There are two approaches for this. 
        1. Restrict default account to localhost, as shown in the following example.  In this case you will need to use another account to run queries between clusters and will need to put it in the cluster definition.  

            ```
            <users>
            <default>
            	<password replace="replace"></password>
            <networks>
            <ip>::1</ip>
            <!-- OR -->
            <ip>127.0.0.1</ip>
            <networks>
            ```


        2. Restrict the default account to hosts belonging to the cluster, as in the following example. 

            ```
            <users>
            <default>
            		<password replace="replace"></password>
            <networks>
                			<host>chi-zdist1-c1-0-0</host>
                			<host>chi-zdist1-c1-0-1</host>
                			<host>chi-zdist1-c1-1-0</host>
                			<host>chi-zdist1-c1-1-1</host>
            <networks>
            ```


    4. Save the file. 
3. Login with clickhouse-client to test that the user or profile is properly defined. ClickHouse picks up new entries in users.d without a restart.


### Removing a user from ClickHouse



1. Remove the corresponding file in /etc/clickhouse-server/users.d. 
2. Login with clickhouse-client to ensure that the user is gone. 


### Changing network settings on Clickhouse **(Restart Required)**

Network settings are stored in /etc/clickhouse-server/config.xml.  Key configuration tags are:



*   &lt;listen_host> -- Listener host, e.g., 0.0.0.0
*   &lt;http_port> -- HTTP port
*   &lt;tcp_port> -- Native TCP/IP port
*   &lt;https_port> -- HTTPS (encrypted) port
*   &lt;tcp_port_secure> -- Encrypted native TCP/IP port

Change these as follows:



1. Edit config.xml and alter network-related tag values.
2. Restart Clickhouse, e.g., **systemctl restart clickhouse-server**
3. Check connectivity using clickhouse-client and curl to ensure ports are open and responsive. 


### Moving a partition between ClickHouse servers (Some Unavailability)

Moving partitions between servers is necessary to rebalance shards or to retire a shard.  The following procedure moves parts by copying the enclosing partition from a source server to a target server, then deleting on the source. 



1. On the source server, select a partition to move.  You can use a query like the following to get the partition name. 

    ```
    SELECT DISTINCT partition
    FROM system.parts
    WHERE (database = 'airline') AND (table = 'ontime')
    LIMIT 5
    ```


2. On the target server use ALTER TABLE FETCH and ATTACH to move the partition to move each partition. Example shown below, where FROM points to the ZooKeeper path to the table. 

    ```
    ALTER TABLE ontime FETCH PARTITION ID '201002' 
  FROM '/clickhouse/tables/0/ontime;
    ALTER TABLE ontime ATTACH PARTITION ID '201002;
    ```


3. On the source server detach the partition using ALTER TABLE DETACH as shown below.  You can then delete it after confirming the partition is visible on the target. 

    ```
    ALTER TABLE ontime DETACH PARTITION ID '201002;
    set allow_drop_detached=1;
    ALTER TABLE table_name DROP DETACHED PARTITION ID '201002';
    ```


4. If partitions contain more than 50GB of data you will need to take special steps to delete the data. 
    1. Increase (or set to zero) max_[table/partition]_size_to_drop in server config and restart ClickHouse. 
    2. Create file /var/lib/clickhouse/flags/force_drop_table and make sure that ClickHouse has write permission for it.  Example of command: 

        ```
        sudo touch '/var/lib/clickhouse/flags/force_drop_table' && sudo chmod 666 '/var/lib/clickhouse/flags/force_drop_table'

        ```


For more information see [ALTER FETCH PARTITION](https://clickhouse.tech/docs/en/sql-reference/statements/alter/partition/#alter_fetch-partition) in the ClickHouse docs. 


### Setting up Zookeeper


#### Preparation

You need to decide if you will run ZooKeeper in standalone or in replicated mode, find servers where you will install ZooKeeper, and enumerate your servers.

Running ZooKeeper in standalone mode is convenient for evaluation, development, and testing. But in production, you should run ZooKeeper in replicated mode. A replicated group of servers is called an ensemble, and in replicated mode, all servers in the ensemble have copies of the same configuration file. For replicated mode, a minimum of three servers are required, and you should always have an odd number of servers. Here are trade-offs between numbers of Zookeeper ensemble sizes.



*   1 server -- Use **<span style="text-decoration:underline;">ONLY</span> **for developer/QA purposes. This should never be used for production environments. 
*   3 servers --  Optimal setup that works even for very heavily loaded systems after proper tuning (e.g. in Yandex).
*   5 servers -- Less likely to lose quorum entirely but having 5 or more servers will result in longer quorum acquisition times. 

Here are some additional important best practices to keep Zookeeper happy. 



*   <span style="text-decoration:underline;">Never</span> deploy even numbers of Zookeeper servers in an ensemble.  
*   Do not install Zookeeper on ClickHouse nodes. 
*   Do not share Zookeeper with other applications like Kafka. 
*   Place the Zookeeper dataDir and logDir on fast storage that will not be used for anything else. 


#### Installation

Install packages



1. zookeeper (3.4.9 or later)
2. netcat


#### Configure ZooKeeper



1. /etc/zookeeper/conf/myid

    The myid file consists of a single line containing only the text of that machine's id. So myid of server 1 would contain the text "1" and nothing else. The id must be unique within the ensemble and should have a value between 1 and 255.

2. /etc/zookeeper/conf/zoo.cfg

    Every machine that is part of the ZooKeeper ensemble should know about every other machine in the ensemble. You accomplish this with a series of lines of the form **server.id=host:port:port**


    ```
    # specify all zookeeper servers
    # The fist port is used by followers to connect to the leader
    # The second one is used for leader election
    server.1=zookeeper1:2888:3888
    server.2=zookeeper2:2888:3888
    server.3=zookeeper3:2888:3888
    ```



    These lines must be the same on every ZooKeeper node

3. /etc/zookeeper/conf/zoo.cfg

    **This setting MUST be added on every ZooKeeper node:**


    ```
    # The time interval in hours for which the purge task has to be triggered.
    # Set to a positive integer (1 and above) to enable the auto purging. Defaults to 0.
    autopurge.purgeInterval=1

    ```



#### Start ZooKeeper

sudo -u zookeeper /usr/share/zookeeper/bin/zkServer.sh


#### Check if ZooKeeper is OK

Commands:


    echo ruok | nc localhost 2181


    echo mntr | nc localhost 2181


    echo stat | nc localhost 2181

Logs: /var/log/zookeeper/zookeeper.log

Snapshots: /var/lib/zookeeper/version-2/


#### Connect to ZooKeeper

bin/zkCli.sh -server 127.0.0.1:2181


#### Tune ZooKeeper

**These settings you might want to change if connections between the nodes are not very reliable: **


```
/etc/zookeeper/conf/zoo.cfg
# The number of ticks that the initial synchronization phase can take
initLimit=10
# The number of ticks that can pass between sending a request and getting an acknowledgement
syncLimit=5
```

**These settings you might want to change depending on your workload, so that snapshots are not created too often:**

```
/etc/zookeeper/conf/zoo.cfg
# To avoid seeks ZooKeeper allocates space in the transaction log file in blocks of preAllocSize kilobytes.
# The default block size is 64M. One reason for changing the size of the blocks is to reduce the block size
# if snapshots are taken more often. (Also, see snapCount).
preAllocSize=65536
# ZooKeeper logs transactions to a transaction log. After snapCount transactions are written to a log file a
# snapshot is started and a new transaction log file is started. The default snapCount is 10,000.
snapCount=10000
```

Documentation:

*   ZooKeeper Getting Started Guide [https://zookeeper.apache.org/doc/current/zookeeperStarted.html](https://zookeeper.apache.org/doc/current/zookeeperStarted.html)
*   ClickHouse tips: [https://clickhouse.yandex/docs/en/operations/tips/#zookeeper](https://clickhouse.yandex/docs/en/operations/tips/#zookeeper)
*   [https://docs.confluent.io/current/zookeeper/deployment.html](https://docs.confluent.io/current/zookeeper/deployment.html)


### Configuring ClickHouse to use Zookeeper **(Restart Required)**

Follow the steps shown below.  Recommended settings are located [here ](https://clickhouse.tech/docs/en/operations/server-configuration-parameters/settings/#server-settings_zookeeper)in the ClickHouse documentation. 



1. Create a configuration file with the list of ZooKeeper nodes.  Best practice is to put the file in /etc/clickhouse-server/config.d/zookeeper.xml. 

    ```
    <yandex>

            <zookeeper>
                <node>
                    <host>example1</host>
                    <port>2181</port>
                </node>
                <node>
                    <host>example2</host>
                    <port>2181</port>
                </node>
                <session_timeout_ms>30000</session_timeout_ms>
                <operation_timeout_ms>10000</operation_timeout_ms>
                <!-- Optional. Chroot suffix. Should exist. -->
                <root>/path/to/zookeeper/node</root>
                <!-- Optional. Zookeeper digest ACL string. -->
                <identity>user:password</identity>
            </zookeeper>
        </yandex>

    ```



2. Check the distributed_ddl parameter in config.xml. You can redefine this parameter in another configuration file, and you can change the path to any value that you like. If you have several ClickHouse clusters using same zookeeper, then distributed_ddl path should be unique for every ClickHouse cluster setup.

    ```
    <!-- Allow to execute distributed DDL queries (CREATE, DROP, ALTER, RENAME) on cluster.
         Works only if ZooKeeper is enabled. Comment it if such functionality isn't required. -->
    <distributed_ddl>
        <!-- Path in ZooKeeper to queue with DDL queries -->
        <path>/clickhouse/task_queue/ddl</path>

        <!-- Settings from this profile will be used to execute DDL queries -->
        <!-- <profile>default</profile> -->
    </distributed_ddl>
    ```


3. Check /etc/clickhouse-server/preprocessed/config.xml. You should see your changes there.
4. Restart ClickHouse.  Check ClickHouse connection to Zookeeper using documented procedure. 


### Checking ClickHouse connection to Zookeeper

Use this procedure to check connectivity between ClickHouse and Zookeeper. 



1. Confirm that ClickHouse can connect to Zookeeper. You should be able to query the system.zookeeper table, and you should see the path for distributed DDL created in ZooKeeper through that table. If something went wrong, check ClickHouse logs.

    ```
    $ clickhouse-client -q "select * from system.zookeeper where path='/clickhouse/task_queue/'"
    ddl		17183334544	17183334544	2019-02-21 21:18:16	2019-02-21 21:18:16	0	8	0	0	0	8	17183370142	/clickhouse/task_queue/
    ```


2. Confirm Zookeeper accepts connection from ClickHouse. You can also see on ZooKeeper nodes if a connection was established. You should see the IP address of the ClickHouse server in the list of clients:

    ```
    $ echo stat | nc localhost 2181
    Zookeeper version: 3.4.9-3--1, built on Wed, 23 May 2018 22:34:43 +0200
    Clients:
     /10.25.171.52:37384[1](queued=0,recved=1589379,sent=1597897)
     /127.0.0.1:35110[0](queued=0,recved=1,sent=0)

    ```



### Creating a replicated table

Replicated tables use a replicated table engine, for example ReplicatedMergeTree. The following example shows how to create a simple replicated table. It assumes that you have defined appropriate macro values for cluster, shard, and replica in macros.xml. For details consult  [https://clickhouse.tech/docs/en/engines/table-engines/mergetree-family/replication/](https://clickhouse.tech/docs/en/engines/table-engines/mergetree-family/replication/) \
 \
`CREATE TABLE test ON CLUSTER '{cluster}'`


```
(
	timestamp DateTime,
	contractid UInt32,
	userid UInt32
) ENGINE = ReplicatedMergeTree('/clickhouse/tables/{cluster}/{shard}/default/test', '{replica}')
PARTITION BY toYYYYMM(timestamp)
ORDER BY (contractid, toDate(timestamp), userid)
SAMPLE BY userid;
```


The ON CLUSTER clause ensures the table will be created on the nodes of {cluster} (a macro value).  This example automatically creates a ZooKeeper path for each replica table that looks like the following:

/clickhouse/tables/{cluster}/{replica}/default/test, e.g., /clickhouse/tables/c1/0/default/test

You can see Zookeeper data for this replica node with the following query: 

SELECT *

FROM system.zookeeper

WHERE path = '/clickhouse/tables/c1/0/default/test'


### Removing a replicated table

Use DROP TABLE as shown in the following example.  The ON CLUSTER clause ensures the table will be deleted on all nodes.  Omit it to delete the table only on a single node. 


```
DROP TABLE test ON CLUSTER '{cluster}';
```


As each table is deleted the node is removed from replication and the information for the replica is cleaned up.  When no more replicas exist, all Zookeeper data for the table will be cleared.


### Cleaning up Zookeeper data for replicated tables

**_NOTE: Cleaning up Zookeeper data manually can corrupt replication if you make a mistake.  Raise a support ticket and ask for help if you have <span style="text-decoration:underline;">any</span> doubt concerning the procedure._**

Zookeeper data for the table might not be cleared fully if there is an error when deleting the table, or the table becomes corrupted, or the replica is lost. You can clean up Zookeeper data in this case manually using the Zookeeper rmr command.  Here is the procedure: 



1. Login to Zookeeper server. 
2. Run zkCli.sh command to connect to server. 
3. Locate the path to be deleted, e.g.:   \
  `ls /clickhouse/tables/c1/0/default/test`
4. Remove the path recursively, e.g.,  \
  `rmr /clickhouse/tables/c1/0/default/test`

New ClickHouse versions now support [SYSTEM DROP REPLICA](https://clickhouse.tech/docs/en/sql-reference/statements/system/#query_language-system-drop-replica) which is much simpler. 


## Fault Diagnosis and Remediation

Some procedures shown below may have a degree of risk depending on the underlying problem.  These procedures are marked with “Call Support” and include raising a support ticket as the first step. 


### Restarting a crashed ClickHouse server

ClickHouse servers are managed by systemd and normally restart following a crash.  If a server does not restart automatically, follow the steps shown below. 



1. Access the ClickHouse error log for the failed server (located in /var/lib/clickhouse-server/clickhouse-server.err.log). 
2. Seek to the bottom and look for a stack trace showing the cause of the failure. 
3. If there is a stack trace:
    1. If the problem is obvious, fix the problem and run ‘systemctl restart clickhouse-server’ to restart.  Confirm that the server restarts. 
    2. If the problem is not obvious, open an Altinity Support Ticket and provide the error log message. 
4. If there is no stack trace, ClickHouse may have been terminated by the OOM-killer due to excessive memory usage:
    3. Open the most recent syslog file (/var/log/syslog). 
    4. Look for OOM-killer messages.
    5. If found, see “Handling out-of-memory errors” below.
    6. If the problem is not obvious, raise a support ticket and provide a description of the problem. 


### Replacing a failed cluster node



1. Ensure the old node is truly offline and will not return. 
2. Create new node with same macros.xml definitions as previous node. 
3. If possible use same hostname as failed node. 
4. Copy the metadata folder from a healthy replica. 
5. Set the force_restore_data so that ClickHouse wipes out existing Zookeeper information for the node and replicates all data: 


```
       sudo -u clickhouse touch /var/lib/clickhouse/flags/force_restore_data

```



6. Start clickhouse. 
7. Wait until all tables are replicated.  You can check progress using ‘SELECT count(*) FROM system.replication_queue’. 


### Replacing a failed zookeeper node



1. Configure ZooKeeper on a new server
2. Use the same hostname and myid as the failed node if possible. 
3. Start ZooKeeper on the new node, make sure it can connect to the ensemble. 
4. If the new node has a different hostname or myid, modify zoo.cfg on the other nodes of the ensemble and restart them (if ZooKeeper does not support dynamic configuration changes) - ClickHouse’s sessions will be interrupted.
5. Make changes in ClickHouse configuration files if needed. A restart might be required for the changes to take effect.


### Recovering from complete Zookeeper loss **(Call Support)**

Complete loss of Zookeeper is a serious event and should be avoided at all costs by proper Zookeeper management.  Follow this procedure *only* if you have lost all data in Zookeeper as it is time-intensive and will cause affected tables to be unavailable. 



1. Raise a support ticket before taking any steps. 
2. Ensure that Zookeeper is empty and working properly. 
3. Follow the instructions to convert replicated tables to non replicated: [https://clickhouse.tech/docs/en/engines/table-engines/mergetree-family/replication/#converting-from-replicatedmergetree-to-mergetree](https://clickhouse.tech/docs/en/engines/table-engines/mergetree-family/replication/#converting-from-replicatedmergetree-to-mergetree)
4. Once you can see the data, pick one of the non-replicated tables which will be used as a 'healthy' one, and which will be used to resync the data to all other nodes.
5. On that node follow the instructions to change non replicated tables to replicated. [https://clickhouse.tech/docs/en/engines/table-engines/mergetree-family/replication/#converting-from-mergetree-to-replicatedmergetree](https://clickhouse.tech/docs/en/engines/table-engines/mergetree-family/replication/#converting-from-mergetree-to-replicatedmergetree)
6. On the remaining nodes drop MergeTree tables and create a Replicated table once again using the same definition as the healthy table. 
7. ClickHouse will sync from the healthy table to all other tables.  


### Handling problems with ClickHouse connectivity

If an application cannot connect to ClickHouse take the following steps. 



1. Distinguish which interface is involved (HTTP vs. native TCP/IP, encrypted vs. non-encrypted) and the relevant port. 
2. Login to the server host and check local connectivity using clickhouse-client. If you cannot connect, stop here and address the problem. 
    1. Ensure that ClickHouse is running.  
    2. Run ‘netstat -ant |grep LISTEN’ and check that there is a listener on the port you are connecting to.  
    3. If a TCP/IP port, use clickhouse-client, e.g., ‘clickhouse-client [--secure] --port=&lt;port>’
    4. If an HTTP port, use curl, e.g., ‘curl [http://localhost:8123/ping](http://localhost:8123/ping)’ or ‘curl [https://localhost:8123/ping](https://localhost:8123/ping)’
3. If applicable connect to ClickHouse from the remote host that is having problems.  
    5. If a TCP/IP port, use clickhouse-client, e.g., ‘clickhouse-client [--secure] --host=&lt;host/IP address> --port=&lt;port>’
    6. If an HTTP port, use curl, e.g., ‘curl http://host_or_ip_address:8123/ping’ or ‘curl [https://host_or_ip_address:8123/ping](https://host_or_ip_address:8123/ping)’
    7. If the connection immediately fails check ClickHouse connectivity to ensure it is listening on an external interface like 0.0.0.0.  If the connection hangs, check firewall or security group settings. 


### Handling authentication failures

To diagnose an authentication failure, perform the following.



1. Replicate the problem from the command line.  
    1. For clients that use native TCP/IP protocol, try to login using clickhouse-client.  Example: clickhouse-client --user=myuser --password=secret
    2. For clients that use the HTTP interfaces, try submitting a request using curl with basic authentication.  Example: <code> curl --user myuser:secret [http://localhost:8123/?query=select%201](http://localhost:8123/?query=select%201)</code>
2. Check the ClickHouse system and error log for messages. 


### Handling read-only tables

Read-only tables occur when ClickHouse cannot access Zookeeper to record inserts on a replicated table. 



1. Login with clickhouse-client. 
2. Execute the following query to confirm that ClickHouse can connect to Zookeeper. \
`$ clickhouse-client -q "select * from system.zookeeper where path='/'"` \
This query should return one or more ZNode directories. 
3. Execute the following query to check the state of the table.  \
`SELECT * from system.replicas where table='table_name'`
4. If there are connectivity problems, check the following.
    1. Ensure &lt;zookeeper> tag in ClickHouse configuration has the correct Zookeeper host names and ports.
    2. Ensure that Zookeeper is running. 
    3. Ensure that Zookeeper is accepting connections.  Login to the Zookeeper host and try to connect using zkClient.sh. 


### Handling “too-many parts” errors

An error like the following indicates ClickHouse is having problems with background merging of data: DB::Exception: Too many parts (412). Merges are processing significantly slower than inserts. 

Too-many parts errors may arise for a wide variety of reasons, so there’s no single cause.  Here are some suggestions to investigate to help with diagnosis.  In our experience the last cause (#3) is very common. 



1. Are merges working properly?  
    1. Try to merge manually using the steps below. 
        1. Find the table that is causing the issue:

            ```
            select 
  database, table, partition, countIf(active), count()
            from system.parts 
            group by database, table, partition
            order by countIf(active) desc limit 10
            ```


        2. Try to merge manually in ClickHouse to find out why it is not working as shown in the following example. (‘2019-10-07’ is the part name.)

            ```
            set optimize_throw_if_noop=1;
            optimize table table_name PARTITION '2019-10-07' FINAL
            ```


    2. Monitor pending merges on the table by selecting from system.merges, which tracks merge activity on all tables.  
    3. Ensure that replication is operating on the table by selecting from system.replication_queue. 
2. Is there an unexpected peak or change in the insert rate? 
    4. Stop inserts temporarily to let ClickHouse complete merges and catch up. 
    5. If ClickHouse is just temporarily overloaded, you can increase the numbers of parts that ClickHouse will allow before throwing an error as shown below.  This requires a server restart. 

        ```
        <yandex>
          <merge_tree>
            <parts_to_throw_insert>600</parts_to_throw_insert>
          </merge_tree>
        </yandex>
        ```


3. Were there recent changes to the system? 
    6. Look for schema changes that might cause ClickHouse to become slower. 
    7. Look for application changes:
        3. Differences in the number or size of INSERTs. 
        4. Changes in query patterns that may create additional load and contention with INSERT traffic
    8. Look for changes in the underlying infrastructure such as changes in CPU capacity, storage type, network capacity, etc.  

If the problem is not obvious, open a support ticket. 


### Handling Problems Related to Moving Data

Moving ClickHouse data at the file system level must be done carefully and only using documented [ALTER TABLE commands](https://clickhouse.tech/docs/en/sql-reference/statements/alter/partition/) to detach, attach, move, or fetch partitions.  If you move files manually outside these commands, it may confuse ClickHouse about state of tables or databases resulting in errors like the following:


```
DROP DATABASE foo ON CLUSTER `{cluster}`
...
Received exception from server (version 20.3.12):
Code: 1000. DB::Exception: Received from localhost:9440. DB::Exception: There was an error on [ch-event01-2:9440]: Poco::Exception. Code: 1000, e.code() = 39, e.displayText() = Directory not empty: /data/clickhouse/data/foo (version 20.3.12.112 (official build)).
```


Procedures to avoid/correct this problem are as follows. 



1. Only use documented ALTER TABLE commands to move partitions. See the section on “Moving a partition between ClickHouse servers” for an example. 
2. If in doubt about procedures, contact Altinity support about correct steps to move data. 
3. If an error arises, contact Altinity Support to help with cleanup. 


### Diagnosing slow query performance

Slow query performance may arise from a wide range of issues.  Here are some generation recommendations. 



1. If possible, isolate the query that is causing problems.   
2. Run the query in clickhouse-client with the --send_logs_level=’trace’ option set.  This will dump system trace messages for any query you run. 
3. Inspect the trace log output from the query to determine how many threads are being used, whether the query takes advantage of indexes or filter conditions to skip reading marks, and how much data the query is reading. 
4. Look for any of the following common causes of slow performance.
    1. Recent upgrades that changed table schema or the query itself.
    2. Addition of large amounts of data that make the query more costly to run. 
    3. Poor compression on columns in the query.  (Select the table columns in system.columns to check compression levels.)

If the problem is not obvious, open a support ticket. 
