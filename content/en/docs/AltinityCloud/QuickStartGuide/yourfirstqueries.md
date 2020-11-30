---
title: "Your First Queries"
linkTitle: "Your First Queries"
weight: 5
description: >
    How to use Cluster Explore to run queries, view table structures and available processes.
---

Once your cluster is created, time to create some tables and do some queries.

 

For those experienced with ClickHouse, this will be very familiar.  For people who are new to ClickHouse, creating tables is very similar to other SQL based databases, with some extra syntax that defines the type of table we’re making.  This is part of what gives ClickHouse its speed.  For complete details on ClickHouse commands, see the[ ClickHouse SQL Reference Guide](https://clickhouse.tech/docs/en/sql-reference/syntax/).


### **Cluster Explore**

The Cluster Explore page allows you to run queries, view the schema, and check on processes for your cluster.  It’s a quick way of getting into your ClickHouse database, run commands and view your schema straight from your web browser.  We’ll be using this to generate our first tables and input some data.

 

To access Cluster Explore for your cluster, just click **Explore** for the specific cluster to manage.

 

For our example, we’re going to create two tables:

 



*   **events_local**:  This table will use the ReplicatedMergeTree table engine.  If you don’t know about table engines, don’t worry about that for now.  See the[ ClickHouse Engines pag](https://clickhouse.tech/docs/en/engines/table-engines/)e for complete information.
*   **events**: This table will be distributed on your cluster with the Distributed table engine.

 

In our examples, we’ll be using macro variables - these are placed between curly brackets and let us use the same SQL commands on different clusters and environments without having to fill in every detail.  Any time you see an entry like `{cluster}` or `{shard}` you should recognize those as a macro variable.

 

The commands below will create these tables into the **default** database on your cluster.  


### **Create Tables**

 

To create your first tables:

 



1. From the **Altinity Cloud Manager** select **Explore** for the cluster to manage**.**
2. Select **Query**.
3. For our first table, copy and paste the following into the **Query** window, then select **Execute**.

    ```
CREATE TABLE IF NOT EXISTS events_local ON CLUSTER '{cluster}' (
    event_date  Date,
    event_type  Int32,
    article_id  Int32,
    title       String
) ENGINE = ReplicatedMergeTree('/clickhouse/{cluster}/tables/{shard}/{database}/{table}', '{replica}')
    PARTITION BY toYYYYMM(event_date)
    ORDER BY (event_type, article_id);
```



 


    You should see the following under **Execute** confirming the commands ran, just replacing **docdemo** with your cluster:


```
docdemo.demo.beta.altinity.cloud:8443 (query time: 0.342s)
chi-docdemo-docdemo-0-0	9000 0	 	1	0
chi-docdemo-docdemo-0-1	9000 0	 	0	0
```


 



4. Now let’s create our second table.  Back in the **Query** window, enter the following and select **Execute**:

    ```
CREATE TABLE events ON CLUSTER '{cluster}' AS events_local
    ENGINE = Distributed('{cluster}', default, events_local, rand())
```



    Once again, you should see confirmation under **Execute**:


    ```
docdemo.demo.beta.altinity.cloud:8443 (query time: 0.162s)
chi-docdemo-docdemo-0-0	9000 0	 	1	0
chi-docdemo-docdemo-0-1	9000 0	 	0	0
```


5. Now that we have some tables, let’s not leave them empty.  Inserting data into a ClickHouse table is very similar to most SQL systems.  Let’s Insert our data, then do a quick Select on it.  Enter the following Insert command into **Query**, then select **Execute**:

    ```
INSERT INTO events VALUES(today(), 1, 13, 'Example');
```



    You’ll see the results confirmed under **Execute**, just like before.


 


```
OK.
```



    Then enter the following **Select** command and select **Execute** again: \



```
SELECT * FROM events;
```



    The results will look like the following:


```
docdemo.demo.beta.altinity.cloud:8443 (query time: 0.018s)
┌─event_date─┬─event_type─┬─article_id─┬─title───┐
│ 2020-11-19 │          1 │         13 │ Example │
└────────────┴────────────┴────────────┴─────────┘
```



### **View Schema**

The Database Schema shows a graphical view of your cluster’s database, the tables in it, and their structure.  

 

To view your Schema:

 



1. From the **Altinity Cloud Manager** select **Explore** for the cluster to manage**.**
2. Select **Schema**.

 

You can expand the databases to display the tables in each database, or select the table to view its details, schema, and some sample rows. 


### **View Processes**

To view current actions running on your cluster select **Processes**.  This displays what processes are currently running, what user account they are running under, and allows you to view more details regarding the process.