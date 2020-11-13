---
title: System Monitoring
layout: default
published: true
order: 4
---
## Monitoring Frameworks

Prometheus combined with Grafana is overall the most frequently used framework for ClickHouse monitoring.  Please see the [ClickHouse Monitoring 101 Webinar](https://www.altinity.com/webinarspage/2020/04/01/clickhouse-monitoring-101-what-to-monitor-and-how) for complete details on deployment. 

Other commonly used frameworks include Instana, Zabbix, and ELK stack. Customers have used all of these successfully. 


## ClickHouse


### Handy Status Checks and Alert Conditions

The following status checks show quick checks that can be used to detect problems with ClickHouse.  We use these regularly in support calls.  They are also good targets for monitoring. 


<table>
  <tr>
   <td><strong>Check Name</strong>
   </td>
   <td><strong>Shell or SQL command</strong>
   </td>
   <td><strong>Severity</strong>
   </td>
  </tr>
  <tr>
   <td>ClickHouse server is up.
   </td>
   <td>
    `$ curl 'http://localhost:8123/'`
    `Ok.`
   </td>
   <td>`Critical`
   </td>
  </tr>
  <tr>
   <td>Too many simultaneous queries. Maximum: 100
   </td>
   <td>`select value from system.metrics `
`where metric='Query'`
   </td>
   <td>`Critical`
   </td>
  </tr>
  <tr>
   <td>Replication status
   </td>
   <td>`$ curl 'http://localhost:8123/replicas_status'`
`Ok.`
   </td>
   <td>`High`
   </td>
  </tr>
  <tr>
   <td>Read only replicas (reflected by `replicas_status` as well)
   </td>
   <td>`select value from system.metrics `
`where metric='ReadonlyReplica'`
   </td>
   <td>`High`
   </td>
  </tr>
  <tr>
   <td>Some replication tasks are stuck
   </td>
   <td>`select count()`
`from system.replication_queue`
`where num_tries > 100`
   </td>
   <td>`High`
   </td>
  </tr>
  <tr>
   <td>ZooKeeper is available
   </td>
   <td>`select count() from system.zookeeper `
`where path='/'`
   </td>
   <td>`Critical for writes`
   </td>
  </tr>
  <tr>
   <td>ZooKeeper exceptions
   </td>
   <td>`select value from system.events `
`where event='ZooKeeperHardwareExceptions'`
   </td>
   <td>`Medium`
   </td>
  </tr>
  <tr>
   <td>Ensure CH nodes are available
   </td>
   <td>`$ for node in `echo "select distinct host_address from system.clusters where host_name !='localhost'" | curl 'http://localhost:8123/' --silent --data-binary @-`; do curl "http://$node:8123/" --silent ; done | sort -u`
`Ok.`
   </td>
   <td>`High`
   </td>
  </tr>
  <tr>
   <td>Ensure all CH clusters are available (i.e. every configured cluster has enough replicas to serve queries)
   </td>
   <td>`for cluster in `echo "select distinct cluster from system.clusters where host_name !='localhost'" | curl 'http://localhost:8123/' --silent --data-binary @-` ; do clickhouse-client --query="select '$cluster', 'OK' from cluster('$cluster', system, one)" ; done `
   </td>
   <td>`Critical`
   </td>
  </tr>
  <tr>
   <td>There are files in 'detached' folders
   </td>
   <td>`select count() from system.detached_parts`
   </td>
   <td>`Medium`
   </td>
  </tr>
  <tr>
   <td>Too many parts problems, including: \
Number of parts is growing; \
Inserts are being delayed; \
Inserts are being rejected
   </td>
   <td>`select value from system.asynchronous_metrics `
`where metric='MaxPartCountForPartition';`
`select value from system.events/system.metrics `
`where event/metric='DelayedInserts'; \
select value from system.events `
`where event='RejectedInserts'`
   </td>
   <td>`Critical`
   </td>
  </tr>
  <tr>
   <td>Dictionaries: exception
   </td>
   <td>`select concat(name,': ',last_exception) `
`from system.dictionaries`
`where last_exception != ''`
   </td>
   <td>`Medium`
   </td>
  </tr>
  <tr>
   <td>Time since last ClickHouse server restart.
   </td>
   <td>`select uptime();`
`select value from system.asynchronous_metrics `
`where metric='Uptime'`
   </td>
   <td>`(None)`
   </td>
  </tr>
  <tr>
   <td>DistributedFilesToInsert should not be always increasing
   </td>
   <td>`select value from system.metrics `
`where metric='DistributedFilesToInsert'`
   </td>
   <td>`Medium`
   </td>
  </tr>
  <tr>
   <td>A data part was lost
   </td>
   <td>`select value from system.events `
`where event='ReplicatedDataLoss'`
   </td>
   <td>`High`
   </td>
  </tr>
  <tr>
   <td>Data parts are not the same on different replicas
   </td>
   <td>`select value from system.events where event='DataAfterMergeDiffersFromReplica'; \
select value from system.events where event='DataAfterMutationDiffersFromReplica'`
   </td>
   <td>`Medium`
   </td>
  </tr>
</table>



### Common Metrics

Altinity includes the following system table properties in its internal metrics dashboards.  


<table>
  <tr>
   <td><strong>System Table</strong>
   </td>
   <td><strong>Property</strong>
   </td>
   <td><strong>Description</strong>
   </td>
  </tr>
  <tr>
   <td>asynchronous_metrics
   </td>
   <td>MaxPartCountForPartition
   </td>
   <td>
   </td>
  </tr>
  <tr>
   <td>asynchronous_metrics
   </td>
   <td>ReplicasMaxAbsoluteDelay
   </td>
   <td>Delay between replicas. Persistent delays should be investigated.
   </td>
  </tr>
  <tr>
   <td>asynchronous_metrics
   </td>
   <td>ReplicasMaxInsertsInQueue
   </td>
   <td>Number of pending inserted parts for replication
   </td>
  </tr>
  <tr>
   <td>asynchronous_metrics
   </td>
   <td>ReplicasMaxMergesInQueue
   </td>
   <td>Number of pending merges for replication
   </td>
  </tr>
  <tr>
   <td>asynchronous_metrics
   </td>
   <td>UncompressedCacheBytes
   </td>
   <td>
   </td>
  </tr>
  <tr>
   <td>events
   </td>
   <td>CompressedReadBufferBytes
   </td>
   <td>Uncompressed bytes read from compressed sources (files, network)
   </td>
  </tr>
  <tr>
   <td>events
   </td>
   <td>InsertedBytes
   </td>
   <td>Number of bytes inserted
   </td>
  </tr>
  <tr>
   <td>events
   </td>
   <td>InsertedRows
   </td>
   <td>Number of rows inserted
   </td>
  </tr>
  <tr>
   <td>events
   </td>
   <td>InsertQuery
   </td>
   <td>Number of INSERT statements
   </td>
  </tr>
  <tr>
   <td>events
   </td>
   <td>MarkCacheHits
   </td>
   <td>Number of marks found in cache
   </td>
  </tr>
  <tr>
   <td>events
   </td>
   <td>MarkCacheMisses
   </td>
   <td>Number of marks that had to be fetched from storage
   </td>
  </tr>
  <tr>
   <td>events
   </td>
   <td>Merge
   </td>
   <td>Number of merge operations
   </td>
  </tr>
  <tr>
   <td>events
   </td>
   <td>MergedRows
   </td>
   <td>Number of rows merged
   </td>
  </tr>
  <tr>
   <td>events
   </td>
   <td>Query
   </td>
   <td>Number of queries to be interpreted and potentially executed
   </td>
  </tr>
  <tr>
   <td>events
   </td>
   <td>ReadCompressedBytes
   </td>
   <td>Number of bytes read from compressed sources
   </td>
  </tr>
  <tr>
   <td>events
   </td>
   <td>RejectedInserts
   </td>
   <td>Number of delayed or rejected inserts. Investigate if growing
   </td>
  </tr>
  <tr>
   <td>events
   </td>
   <td>SelectQuery
   </td>
   <td>Same as Query, but only for SELECT queries
   </td>
  </tr>
  <tr>
   <td>ZookeeperHardwareExceptions
   </td>
   <td>ZookeeperHardwareExceptions
   </td>
   <td>Number of serious Zookeeper errors. Persistent values over 0 should be checked
   </td>
  </tr>
  <tr>
   <td>ZookeeperTransactions
   </td>
   <td>ZookeeperTransactions
   </td>
   <td>Number of transactions on Zookeeper ensemble
   </td>
  </tr>
  <tr>
   <td>metrics
   </td>
   <td>DistributedFilesToInsert
   </td>
   <td>Pending inserts through distributed tables
   </td>
  </tr>
  <tr>
   <td>metrics
   </td>
   <td>LeaderReplica
   </td>
   <td>Number of leader replicas; Will be discontinued in version 20.5.
   </td>
  </tr>
  <tr>
   <td>metrics
   </td>
   <td>MemoryTracking
   </td>
   <td>Number of bytes of allocated memory
   </td>
  </tr>
  <tr>
   <td>metrics
   </td>
   <td>ReadonlyReplica
   </td>
   <td>Number of read-only replicates. Check any number over 0
   </td>
  </tr>
</table>



## Hosts

Basic metrics for hosts should be monitored at a minimum, including:



*   CPU utilization
*   Total and free memory
*   Total and available storage space and number of inodes
*   I/O rates on network and storage