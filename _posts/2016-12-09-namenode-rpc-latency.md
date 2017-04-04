---
layout: post
title:  "NameNode RPC Latency"
date:   2016-12-09 16:56:28 +0530
categories: namenode hdfs performance troubleshooting
---

This article talks about the following style of an alert sent by Cloudera Manager for a HDFS service.

> Dec 6 9:29 PM - RPC Latency Concerning
The health test result for NAME_NODE_RPC_LATENCY has become concerning:
The moving average of the RPC latency is 2.2 second(s) over the previous 5 minute(s).
The moving average of the queue time is 634 millisecond(s).
The moving average of the processing time is 1.5 second(s).
Warning threshold: 1 second(s).

RPC Operations and Locks
------------------------

The NameNode's RPC protocols are typically divided into two categories, each served by its own port:

* **Client RPCs** All operations that is typically done as I/O work by HDFS clients
* **Service RPCs** All remaining cluster-side maintenance operations, such as those required by DataNodes and the FailoverController daemons

The set of `Client RPCs` in their `Cloudera Manager` metric naming style are semi-exhaustively:

* **create** Create a new file entry
* **mkdirs** Create a new directory entry
* **add_block** Add a new block to an open file
* **append** Reopen a closed file for append
* **set_replication** Change the replication factor of a file
* **set_permission** Change the permission of a file
* **set_owner** Change the ownership of a file
* **abandon_block** Abandon a currently open block pipeline
* **complete** Close an open file
* **rename / rename2** Rename a file
* **delete** Delete a file
* **get_listing** List a directory
* **renew_lease** Renew the lease on an open file
* **recover_lease** Recover the lease on an open file from another client
* **get_file_info** Get the stat of a file or directory
* **get_content_summary** Get the stat of a file or directory, with detailed summary of its space utilization and quota counts
* **fsync** Reflect the new length of an open file immediately
* **get_additional_datanode** Add a new DataNode location to existing block pipeline
* **update_block_for_pipeline** Update the block statistics of an existing block pipeline (such as after a failure occurs)
* **update_pipeline** Update the state of an existing pipeline (such as after a failure occurs)
* **get_block_locations** Returns the locations of a given block ID

Upon close observation of the above list, you can note that almost all of these operations operate only on the level of a single file or directory (meaning they are to be constant in operational length), with the exclusion of `get_listing` and `get_content_summary`, which are required to give details and summaries of an unknown number of children under the passed file.

However, for `get_listing` and `get_content_summary` there are additional features in their implementation that can still help provide constant-runtime guarantees. For ex. `get_listing` has a maximum results number returned per request (similar to paging a dataset) which the client has to repeat over until it gets all the desired entries. Similarly, `get_content_summary` has constant-runtime too, but is applied slightly differently as its result is not to be computed at the client-side. For this we'll need to talk of locks.

The NameNode's namespace (the filesystem tree maintained in its memory, represented by the class `org.apache.hadoop.hdfs.server.namenode.FSNamesystem`) uses a `READ`**/**`WRITE` lock model that, while better than a *monolithic* lock, is still not too coarse (such as over a specific path inside the tree).

What this means is that today, for all operations:
* If the operation is `READ` category, it may run in parallel with other `READ` category operations
* If the operation is `WRITE` category, it may only run exclusively, and no `READ` operations may run at this time

Here's the Client RPCs list again, this time with the category mentioned and their runtime complexity:

Client RPC | Lock Type | Constant? | Description
-----------|-----------|-----------|------------
**create** | WRITE | Yes | Create a new file entry
**mkdirs** | WRITE | Yes | Create a new directory entry
**add_block** | WRITE | Yes | Add a new block to an open file
**append** | WRITE | Yes | Reopen a closed file for append
**set_replication** | WRITE | Yes | Change the replication factor of a file
**set_permission** | WRITE | Yes | Change the permission of a file
**set_owner** | WRITE | Yes | Change the ownership of a file
**abandon_block** | WRITE | Yes | Abandon a currently open block pipeline
**complete** | WRITE | Yes | Close an open file
**rename / rename2** | WRITE | Yes | Rename a file
**delete** | WRITE | Yes | Delete a file
**get_listing** | READ | No | List a directory
**renew_lease** | READ | Yes | Renew the lease on an open file
**recover_lease** | WRITE | Yes | Recover the lease on an open file from another client
**get_file_info** | READ | Yes | Get the stat of a file or directory
**get_content_summary** | READ | No | Get the stat of a file or directory, with detailed summary of its space utilization and quota counts
**fsync** | WRITE | Yes | Reflect the new length of an open file immediately
**get_additional_datanode** | READ | Yes | Add a new DataNode location to existing block pipeline
**update_block_for_pipeline** | WRITE | Yes | Update the block statistics of an existing block pipeline (such as after a failure occurs)
**update_pipeline** | WRITE | Yes | Update the state of an existing pipeline (such as after a failure occurs)
**get_block_locations** | READ/WRITE | Yes | Returns the DataNode hostnames that hold the replica of a given block ID

Code-wise, a lock-unlock operation would look like this the below.

For READ locks:

```java
try {
  // The below will block iff there's a WRITE lock being held
  readLock();
  // The below will now be one of the many READ operations executing on the server
  doOperation(RPC Request);
} finally {
  readUnlock();
}
```

For WRITE locks:

```java
try {
  // The below will block if there's any READ lock or WRITE lock being held
  writeLock();
  // The below will now be the only operation executing on the server until we free the lock
  doOperation(RPC Request);
} finally {
  writeUnlock();
}
```

Continuing **get_content_summary** point, the pattern of its implementation is such as below:

```java
while not done {
  try {
    readLock();
    doSmallPartOfTheOperationIncrementally(RPC Request);
  } finally {
    readUnlock();
  }
}
```

The advantage gained by this approach is that the read lock is not held for the entire operation. For ex. if the `get_content_summary` call is run on a directory with 6000 files, we may do 6 iterations of 1000 files each within one lock request. This allows other waiting WRITE calls in middle of the call to grab their lock and give it back to the loop, although now the loop takes longer as it goes back into the wait of the `readLock()` in the each iteration.

One other naturally derived difference between the READ and WRITE categories is that the former does not need to add an entry to the NameNode's edit logs (a write-after file used for durability of all mutations done to the namespace). This means that any READ operation will be an operation that does socket I/O (reading the request and writing the response), memory and CPU work, but does not touch the disk I/O (because it does not need to write anything durably to a file). Conversely, all WRITE operations will want to do all of the earlier and also write durably to the edit log file.

There's one small exclusion to the READ operation rule of not touching the disk - the `get_block_locations` call. When a block location is being requested it is assumed to be equivalent to a file read and as a result of that the access time is updated depending on a precision factor over the file the block belongs to.

Something like this occurs:

```java
if ((fileOfBlock.accessTime - now) > 1h) {
  try {
    writeLock();
    updateAccessTime(fileOfBlock, now);
    doOperation(RPC Request);
  } finally {
    writeUnlock();
  }
} else {
  try {
    readLock();
    doOperation(RPC Request);
  } finally {
    readUnlock();
  }
}
```

This means that the *get_block_locations* may sometimes hold the WRITE lock and add access time mutation entries into the NameNode edit log, if the access time feature is in use (Controlled via *dfs.namenode.accesstime.precision* NameNode hdfs-site.xml property, defaults to enabled, with 1h precision).

The following time-series chart query in Cloudera Manager will show the amount of processing time taken by each of such requests:

```sql
SELECT add_block_avg_time,
       create_avg_time,
       append_avg_time,
       set_replication_avg_time,
       set_permission_avg_time,
       set_owner_avg_time,
       abandon_block_avg_time,
       add_block_avg_time,
       get_additional_datanode_avg_time,
       complete_avg_time,
       rename_avg_time, rename2_avg_time,
       delete_avg_time,
       mkdirs_avg_time,
       get_listing_avg_time,
       renew_lease_avg_time,
       recover_lease_avg_time,
       get_file_info_avg_time,
       get_content_summary_avg_time,
       fsync_avg_time,
       update_block_for_pipeline_avg_time,
       update_pipeline_avg_time,
       get_block_locations_avg_time
WHERE roleType = NAMENODE
```

The metric includes all time from the point the request begins (accepted for handling from queue), till its completed entirely (i.e. responded back to the client). This includes blocking wait times such as lock waits, I/O waits on disks for durable operations, etc.

To summarize what we covered above:
* Client RPCs are designed to be always or mostly constant-like in amount of runtime, by their simple nature or by implementation detail.
* Locking patterns of READ and WRITE category of RPCs.
* READ type operations do not do disk I/O on the NameNode, except `get_block_locations` which may sometimes do it.
* WRITE type operations always do disk I/O on the NameNode.

Moving next to the other category of RPCs, the `Service RPCs`, a semi-exhaustive list of it would be:

* **service_block_received_and_deleted** - Incremental block report of blocks added or deleted since the last report
* **service_send_heartbeat** - Periodic heartbeat of DN health, and command status exchanges
* **service_block_report** - Full block report of all blocks on the DN and their states
* **service_register_datanode** - Necessary one-time RPC of a new DN instance before it can do the others above
* **service_monitor_health** - Health check of NN RPC responsiveness for HA Monitor (ZKFC)

By the very wording of *blocks* in the descriptions above, its clear that not all of these calls are as constant-like as the `Client RPCs` were. Their runtime would depend on the number of blocks or commands being passed around. This now changes the locking pattern we'd explored earlier slightly, in that if any of these are WRITE operations then the amount of time their WRITE lock gets held will now be considerably higher and block everything else.

Here's again a table with the lock types and runtime complexity, as well as their periodicity:

Service RPC | Lock Type | Constant? | Periodicity | Description
------------|-----------|-----------|-------------|------------
**service_block_received_and_deleted** | WRITE | No | 3s | Incremental block report of blocks added or deleted since the last report
**service_send_heartbeat** | READ | No | 3s | Periodic heartbeat of DN health, and command status exchanges
**service_block_report** | WRITE | No | 6h | Full block report of all blocks on the DN and their states
**service_register_datanode** | WRITE | Yes | Every DN or NN restart | Necessary one-time RPC of a new DN instance before it can do the others above
**service_monitor_health** | NO LOCK | Yes | 1s | Health check of NN RPC responsiveness for HA Monitor (ZKFC)

Since the same *namesystem lock* is shared between both the RPC types (which is natural because blocks belong to the same files that are in the filesystem tree) the consequence can be that for very long runs of service RPCs due to higher number of blocks to process per request, there can be blockage on serving all Client RPCs in parallel. This would be very concerning, so there are certain implementation details again to the rescue.

For `service_send_heartbeat`, the commands exchanged are of various types, but among the most varying ones are the commands to replicate existing block replicas (to heal under-replicated blocks) and to delete block replicas (to recover storage after file deletes, or to counter over-replicated blocks). These two types of commands are throttled based on some scaling parameters, such that even if 1 million block replicas are decided to be taken action upon (such as due to a large recursive directory delete), we may only send a finite fraction of the action commands to the DN per heartbeat, checking its status in the next heartbeat before we send it more. To some degree this controls the amount of total time it may take to completely process each such RPC.

For `service_block_received_and_deleted`, which is done every time a heartbeat is, the assumption is that the periodicity of the requests generate some finite upper limit of blocks (i.e. maximum number of changes that a DN can possibly do in 3s). There's no designed hard-limit on this nor a throttle but because of the high heartbeat frequency the number of blocks generated per request is proportionally constant.

Finally, for the `service_block_report` RPC, which sends **all** blocks, we apply a simple limiting logic: If the total number of blocks on the DataNode is less than 1 million, counting across all disks of the DataNode, then we send the whole list as a single request. Otherwise, we divide the requests into pieces equal to each disk's individual block lists. For ex., if across 12 disks there are only a total of 600,000 blocks then only one RPC goes out. If the block count across 12 disks is 6,000,000 instead with each disk carrying around 500,000 blocks individually, then 12 RPCs are made in serial fashion, with each carrying 1 whole disk's listing (5,000,000 blocks).

Since the RPCs are broken out now, the time the lock is held per RPC on the NameNode side is also reduced, helping control the impact - although one can still see a gaping hole here: What if the no. of blocks per disk exceeds 1 million? Isn't that situation worse than the single RPC maximum threshold? The answer is Yes but the fact that there are over 1 million blocks per disk is itself a red alert: There's a small files problem on the cluster (1 million actual full blocks of 128 MB each would total to 128 PB, which is far higher than what a disk may accommodate, so the conclusion would be that each of the 1 million files on the disk are far lesser than 128 MB in size).

The below Cloudera Manager time-series query will show the average of processing time registered for each of these types of operations:

```sql
SELECT service_block_received_and_deleted_avg_time,
       service_send_heartbeat_avg_time,
       service_block_report_avg_time,
       service_register_datanode_avg_time,
       service_monitor_health_avg_time
WHERE roleType = NAMENODE
```

The metric includes all time from the point the request begins (accepted for handling from queue), till its completed entirely (i.e. responded back to the client). This includes blocking wait times such as lock waits. Service RPCs do not make any mutations in the edit logs mostly because they only deal with changes to availability of a block replica and not file metadata. This means that none of the above operations touch the disk I/O for durable writes.

RPC Queues and Handler Threads
------------------------------

When any RPC request arrives at the NameNode after the socket connection is established it is read from the socket and placed into a large but bounded queue called the `RPC Queue`. The fully formed request waits here (in-memory) for something to pick it up and do the requested work plus write back a response on the same socket it came in on. This something is referred to as the `handler`. Ideally a request may not spend any significant time in the RPC queue. If it spends a lot of time there, the indication is clear: A free handler was not available to pick up and operate on the request.

The NameNode runs two ports (one for each type of RPCs, discussed above) and one set of finite handler thread-pool per port. The no. of threads run per such thread-pool is the number of handlers available to serve queued requests arriving on that port.

When a handler polls the queue (which it does continually if it has no current work), it grabs any available entry in it and begins the RPC operation along with the lock pattern that was discussed earlier. The handler may be occupied by the operation its doing for a period of time (time spent waiting for the lock to be acquired, or to run the actual request over the CPU and Memory, or to serialize data back over the socket for response, etc.) and at this time it cannot pick up another request. If all such handlers for a port are busy on various parts of each request, then the time spent by further requests in the queue may rise on an average.

The average processing time of all RPCs (across two ports), as handled by all its parallel handlers, is given by the Cloudera Manager time-series query of:

```sql
SELECT rpc_processing_time_avg_time,
       service_rpc_processing_time_avg_time
WHERE roleType = NAMENODE
```

Similarly, the average time spent by any request in the queue, across all request types, is given by the Cloudera Manager time-series query of:

```sql
SELECT rpc_queue_time_avg_time,
       service_rpc_queue_time_avg_time
WHERE roleType = NAMENODE
```

Generally speaking, its rare to see the time spent in queue exceed the processing time, given the mostly-finite processing times for almost all of the discussed operations. However, if it is observed that the call queue times beat out the processing time averages by a good constant amount, then the conclusion is that there are too few handlers. If this happens, consider increasing the `handler count` of the various ports, configurable via `NameNode hdfs-site.xml` configuration properties `dfs.namenode.handler.count` and `dfs.namenode.service.handler.count`. In this situation it can also be noticed that the RPC queue size would appear almost as a constantly present or growing line, rather than a spiky one, as visualizable by the below Cloudera Manager time-series query:

```sql
SELECT rpc_call_queue_length,
       service_rpc_call_queue_length
WHERE roleType = NAMENODE
```

Having some free handlers at all times does not harm - the amount of memory consumed by tens or a few hundred extra idle threads is not very high.

The RPC processing time increase instead may be caused by factors limited to the processing capability and times of the RPC handler threads (and would also depend upon the type of operation). Use the charts of the earlier section to analyze those.

Edit Log Disk I/O
-----------------

Filesystem mutating operations such as the subset of Client RPCs may occasionally touch the disk for synchronous and durable I/O to the current edit log file of the NameNode. In HA mode the mutation log is written to a local disk plus the network (onwards to JournalNodes, which write synchronously and durably to its local disk). On Linux these operations use the `fsync` system call which can get pretty slow in some situations.

The amount of data written to disk per durable write is typically very small - in order of a few hundred KBs. To write just this amount of data to disk, even synchronously, shouldn't take more than a few hundred milliseconds. However, if the disk is shared with other services that log other data on it, the disk scheduler can cause the `fsync` write operation to slow down by many factors. Also, if the disk is mounted in `data=ordered` mode, then the whole disk buffer cache is written to disk instead of just the part that's written to the edit log.

Ideally the NameNodes and the JournalNodes must be both be set to use dedicated disks to avoid this problem, as the latency in a synchronous write reflects as latency in `Client RPC` operations. To take it a step even further, store the NameNode image and edits directory separately, by configuring the image configuration of `dfs.namenode.name.dir` to one disk path and `dfs.namenode.edits.dir` to another disk path. The image directory can be on a fairly shared disk, but keep the edits disk exclusive to only the NameNode. This setup helps avoid the buffer-cache `fsync` delays that may arise due to a growing `fsimage` size that gets written at every checkpoint period.

Edit log write latencies can be viewed via the following time-series query in Cloudera Manager:

```sql
SELECT syncs_avg_time WHERE roleType = NAMENODE
```

Further, lower-level disk level latencies can be queried for investigation via other low-level tools such as iostat.

Process Pauses
--------------

The NameNode is a Java process, and as a consequence of that, may suffer from Garbage Collection (GC) pauses at any point. While a lot of work has gone into the NameNode's system to avoid generating objects in a manner that can cause long full-GC collection times, it may still occasionally occur. Long GC pauses inside the JVM are auto-detected and logged by the class `org.apache.hadoop.util.JvmPauseMonitor`. A GC paused aftermath log will appear as below:

> 2016-09-27 01:36:00,585 WARN org.apache.hadoop.util.JvmPauseMonitor:
Detected pause in JVM or host machine (eg GC): pause of approximately 36848ms
GC pool 'ParNew' had collection(s): count=1 time=828ms
GC pool 'ConcurrentMarkSweep' had collection(s): count=1 time=36020ms

The process can also pause for a number of other factors, such as memory allocation hangs (typically avoidable by using a minimum-heap-size setting), network or disk resource wait hangs, and other kernel level scheduling troubles. The `JvmPauseMonitor` catches these as well, and indicates them as a pause with no detected GCs. For ex.:

> 2016-07-04 10:26:36,570 WARN org.apache.hadoop.util.JvmPauseMonitor:
Detected pause in JVM or host machine (eg GC): pause of approximately 81434ms
No GCs detected

Since the RPC processing time metric is a wall-clock time measure, these pauses reflect on their reported averages after the pausing recovers. In such a case, the heap needs to be tuned or the recent operations done (prior to a large full GC), or the lower level resource problems need to be analyzed, depending on what caused the pause.

The following time-series query in Cloudera Manager will also tell if the NameNode has been generally experiencing any pauses (period of pause and its frequency), as a chart:

```sql
SELECT pause_time_rate,
       pauses_rate
WHERE roleType = NAMENODE
```
