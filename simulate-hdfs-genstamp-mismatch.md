# Simulate HDFS GENSTAMP_MISMATCH Error

1. Upload a file to HDFS with 1 replica block.

```
2021-07-15 22:43:01,060 INFO org.apache.hadoop.hdfs.server.datanode.DataNode: Receiving BP-789828614-192.168.0.61-1579328790708:blk_1073764947_24132 src: /192.168.0.61:59394 dest: /192.168.0.63:50010
2021-07-15 22:43:01,108 INFO org.apache.hadoop.hdfs.server.datanode.DataNode.clienttrace: src: /192.168.0.61:59394, dest: /192.168.0.63:50010, bytes: 1048576, op: HDFS_WRITE, cliID: DFSClient_NONMAPREDUCE_-49167975_1, offset: 0, srvID: 7f24777b-a67d-4f3b-ad59-a8c006b115dc, blockid: BP-789828614-192.168.0.61-1579328790708:blk_1073764947_24132, duration(ns): 24159044
2021-07-15 22:43:01,109 INFO org.apache.hadoop.hdfs.server.datanode.DataNode: PacketResponder: BP-789828614-192.168.0.61-1579328790708:blk_1073764947_24132, type=LAST_IN_PIPELINE terminating
```

In this case, the block ID is **blk_1073764947**, genstamp is **24132**.

2. Change the block metadata filename from *blk_1073764947_24132.meta* to *blk_1073764947_25000.meta* then restart the datanode. The genstamp is **25000** now.

3. Save the meta from namenode.

```
45 files and directories, 30 blocks = 75 total
Live Datanodes: 3
Dead Datanodes: 0
Metasave: Blocks waiting for reconstruction: 0
Metasave: Blocks currently missing: 1
/data.1m: blk_1073764947_24132 MISSING (replicas: l: 0 d: 0 c: 1 e: 0)  192.168.0.63:50010(corrupt) : 
Mis-replicated blocks that have been postponed:
Metasave: Blocks being replicated: 0
Metasave: Blocks 0 waiting deletion from 0 datanodes.
Corrupt Blocks:
Block=1073764947        Node=192.168.0.63:50010 StorageID=DS-ba98f222-3d3b-4811-84a9-0a8eae229118       StorageState=NORMAL     TotalReplicas=1 Reason=GENSTAMP_MISMATCH
Metasave: Number of datanodes: 3
192.168.0.62:50010 /R1 IN 21463302144(19.99 GB) 2164469760(2.02 GB) 10.08% 12577845248(11.71 GB) 0(0 B) 0(0 B) 100.00% 0(0 B) Fri Jul 16 21:07:46 PDT 2021
192.168.0.63:50010 /R2 IN 21463302144(19.99 GB) 2165530624(2.02 GB) 10.09% 14784077824(13.77 GB) 0(0 B) 0(0 B) 100.00% 0(0 B) Fri Jul 16 21:07:44 PDT 2021
192.168.0.64:50010 /R3 IN 21463302144(19.99 GB) 2164469760(2.02 GB) 10.08% 14779789312(13.76 GB) 0(0 B) 0(0 B) 100.00% 0(0 B) Fri Jul 16 21:07:46 PDT 2021
```

The namenode is aware of this corrupted block due to **GENSTAMP_MISMATCH**.

4. Run `fsck` on the corrupted block.

```
Connecting to namenode via http://namenode.internal:50070/fsck?ugi=hdfs&blockId=blk_1073764947+&path=%2F
FSCK started by hdfs (auth:SIMPLE) from /192.168.0.61 at Fri Jul 16 21:06:48 PDT 2021

Block Id: blk_1073764947
Block belongs to: /data.1m
No. of Expected Replica: 1
No. of live Replica: 0
No. of excess Replica: 0
No. of stale Replica: 0
No. of decommissioned Replica: 0
No. of decommissioning Replica: 0
No. of corrupted Replica: 1
Block replica on datanode/rack: dn2.internal/R2 is CORRUPT	 ReasonCode: GENSTAMP_MISMATCH
```

To recover the missing block, simply rename the block metadata filename to the original one then restart datanode. A `fsck` output after the fix.

```
Connecting to namenode via http://namenode.internal:50070/fsck?ugi=hdfs&blockId=blk_1073764947+&path=%2F
FSCK started by hdfs (auth:SIMPLE) from /192.168.0.61 at Fri Jul 16 21:17:21 PDT 2021

Block Id: blk_1073764947
Block belongs to: /data.1m
No. of Expected Replica: 1
No. of live Replica: 1
No. of excess Replica: 0
No. of stale Replica: 0
No. of decommissioned Replica: 0
No. of decommissioning Replica: 0
No. of corrupted Replica: 0
Block replica on datanode/rack: dn2.internal/R2 is HEALTHY
```
