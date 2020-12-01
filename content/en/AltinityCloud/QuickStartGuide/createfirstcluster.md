---
title: "Create Your First Cluster"
linkTitle: "Create Your First Cluster"
weight: 4
description: >
    How to create your first ClickHouse cluster with Altinity.Cloud.
---
Time to make your first cluster!  For this example, we’re creating a minimally sized cluster, but you can upgrade your cluster later to make it the exact size you need for your ClickHouse needs. 

To create your first cluster:

1. From the Altinity Cloud Manager page, select **Launch Cluster**.  This starts the **Cluster Launch Wizard**.
2. The first page is **Resources Configuration**, where we set the name, size and authentication for the new cluster.  When finished, click **Next**.  Use the following settings:

| Setting | Value |
|---------|-------|
| **Name** | Whatever name you like.  <br />Cluster names will be used to create the DNS name of the cluster.  Therefore, cluster names must follow DNS name restrictions (letters, numbers, and dashes allowed, periods and special characters are not).  <br />Cluster names must be **lower case**. |
| **Node Type** | Select **m5.large**  <br />This is the size of the node.  This selection gives us a cluster with 2 CPUs and 6.5 Gb RAM.  Recall that we can upgrade this cluster later.|
| **Node Storage** | Set to **30 Gb**.  <br />The size of each Cluster node in Gb (gigabytes).  Each node will have the same storage area. |
| **Number of Volumes** | Set to **1**. <br />Network storage can be split into separate volumes.  Use more volumes to increase query performance. |
| **Volume Type** | Select **gp2 (Not Encrypted)**. <br />Volumes can be either encrypted or unencrypted, depending on your security requirements. |
| **Number of Shards** | Set to **1**. <br /> The shard represents a set of nodes.  Shards can then be replicated to provide increased availability and recovery. |
| **ClickHouse Version** | Select **the most recent Altinity Stable Release**. <br /> Your ClickHouse cluster can use the version that best meets your needs.  Note that **all nodes will run the same ClickHouse version**. |
| **ClickHouse User Name** | Autoset to **admin**. <br />The default administrative user. |
| **ClickHouse User Password** and **Confirm Password** | Set to your security requirements.  Both the ClickHouse User Password and Confirm Password must match. |

3. The next page is High Availability Configuration.  This is where you can set your replication, Zookeeper, and backup options.  Use the following values for your first cluster, then click **Next** to continue:

| Setting | Value |
|---|---|
| **Data Replication** | Set to **Enabled**. <br />Data replication duplicates data across replica clusters for increased performance and availability.  |
| **Number of Replicas** | Set to **2**. <br />Only required if **Data Replication** is **Enabled**. <br />Sets the number of replicas for each cluster shard.
| **Zookeeper Configuration** | Select **Single Node **for development and test clusters.**  **Select** Three Nodes** for production usage. <br />**IMPORTANT NOTE**:  This setting cannot be changed later.  <br />Apache Zookeeper manages synchronization between the clusters.
| **Zookeeper Node Type** | **Default** is selected by default.
| **Node Placement** | Select **Separate Nodes**.
| **Enable Backups** | Set to **Enabled**. |

4. The **Connection Configuration** page determines how to communicate with your new cluster.  Set the following values, then select **Next** to continue:

| **Setting** | **Value** |
|---|---|
| **DNS Name** | This is automatically set based on your cluster name. |
| **Use TLS** | Set to **Enabled**. <br />When enabled, communications with your cluster are encrypted with TLS. |
| **Load Balancer Type** | Select **Altinity Edge Ingress**.  <br />**IMPORTANT NOTE**: This setting requires clients to support SNI (Server Name Indication).  This will require the most current Clickhouse client and Python clickhouse-driver. <br />This setting cannot be changed after the cluster is created. |

5. Last page!  **Review & Launch** lets you double check your settings.  When you’re ready, click **Launch**.

It will take a few minutes before your new cluster is ready, so grab your coffee or tea or other hydrating beverage.  When it’s complete, you’ll see your new cluster with all nodes online and all health checks passed.
