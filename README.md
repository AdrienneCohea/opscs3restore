# Background and Problem Overview

DataStax OpsCenter simplifies the task of backup and restore of data out of a DSE (DataStax Enterprise) clusster a lot through its out-of-the-box feature of [Backup Service](https://docs.datastax.com/en/opscenter/6.5/opsc/online_help/services/opscBackupService.html). Through this service, a user can choose to bakup DSE data to multiple locations, including AWS S3, which becomes a more and more popular choice in today's ever-increasing cloud environment.

When backing up a DSE clsuter keyspace/table to AWS S3, the backup files are organized in S3 in the following structure.

```
mybucket/
    snapshots/
      node-id1/
        sstables/
          MyKeyspace-MyTable-ic-5-Data.db
          ...
          MyKeyspace-MyTable-ic-5-TOC.txt
          MyKeyspace-MyTable-ic-6-Data.db
          ...
        1234-ABCD-2014-10-01-01-00/
          backup.json
          MyKeyspace/schema.json
        1234-ABCD-2014-09-30-01-00/
          backup.json
          MyKeyspace/schema.json
          
       node-id2/
         sstables/
         ...
```

Please note that this is just a "virtual" structure. The actual S3 object is simply a storage blob file with certain naming convention. An example is as below fo 
```
mybucket/snapshots/node-id1/sstables/MyKeyspace-MyTable-ic-5-Data.db
```

## Restore Challenge 

When we use OpsCener Service to restore backup data from S3, behind the scene it utilizes the traditional Cassandra "sstableloader" utility. Simply speaking, OpsCenter server, through datatax-agent on each DSE node, fetches backup data from 
S3 bucket and once it is done, it kicks of "sstableloader" to bulk-loading data into DSE cluster. It repeats the same process until all backup data in S3 bucket has been processed.

This approach has pros an cons: 
- The biggest pro is that it can tolerate DSE topology change, which means that the backup data can be restored to 1) the same cluster without any topology change; 2) the same cluster with some topology change; or 3) a brand new cluster.
- A major downside is that it is going to consume extra disk space (and extra disk and network I/O bandwith) in order to complete the whole process. For a keyspace with replication factor N (N > 1, normally 3 or above), it causes N times of the backup data to be ingested into he cluster. Although over the time, the C* compaction process will address the issue. But still, a lot of data has been transmitted over the network and processed in the system.

In many cases, when there is NO DSE cluster topology change, a much faster approach would be: 
1) Simply copy the backup data to its corresponding DSE node, under the right C* keyspace/table (file system) folder
2) Once the data is copied, run "nodetool refresh" command to pick up the data-to-be-retored in DSE cluster.

# 
