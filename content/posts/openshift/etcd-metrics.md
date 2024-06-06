---
title: "OCP etcd metrics"
date: 2023-04-07
categories:
- openshift
tags:
- openshift
---

### Recommended etcd practices
https://docs.openshift.com/container-platform/4.12/scalability_and_performance/recommended-host-practices.html#recommended-etcd-practices_recommended-host-practices

The histogram_quantile(0.99, rate(etcd_network_peer_round_trip_time_seconds_bucket[2m])) metric reports the round trip time for etcd to finish replicating the client requests between the members. Ensure that it is less than 50 ms.

### Metrics to monitor
https://access.redhat.com/articles/6967785#metrics

#### Monitor Leadership changes:

This is expected as per result of installation/upgrade process or day1/2 operations (as result of Machine Config daemon operations), but we don't expect to see it happening during normal operations.
etcdHighNumberOfLeaderChanges alert can help us to identify that situation.
Prometheus query could also be used (sum(rate(etcd_server_leader_changes_seen_total[2m]))).
If happening during normal operation, I/O and network metrics can help us to identify the root cause.

#### I/O Metrics:

etcd_disk_backend_commit_duration_seconds_bucket with p99 duration less than 25ms  
etcd_disk_wal_fsync_duration_seconds_bucket with p99 duration less than 10ms

#### Network metrics:

etcd_network_peer_round_trip_time_seconds_bucket with p99 duration should be less than 50ms.  
Network RTT latency: Big network latency and packet drops can also bring an unreliable etcd cluster state, so network health values (RTT and packet drops) should be monitored.  
etcd can also suffer poor performance if the keyspace grows excessively large and exceeds the space quota.

#### Some key metrics to monitor are:

etcd_server_quota_backend_bytes which is the current quota limit.  
etcd_mvcc_db_total_size_in_use_in_bytes which indicates the actual database usage after a history compaction.  
etcd_debugging_mvcc_db_total_size_in_bytes which shows the database size including free space waiting for defragmentation.  


An etcd database can grow up to 8 GB.  