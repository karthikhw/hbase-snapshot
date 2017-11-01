# hbase-snapshot
## Export Phoenix tables using HBase Snapshot

  HBase snapshot is the best approach for large import and export data to another cluster but has depend the use case whether feasible or not. 
  
  Snapshot is point-in-time capture, you may not able to export by time series (like a day, week, year data separately) but to achieve this you may need a different approach.
  
  This approach allows you to take a snapshot of a table without too much impact on Region Servers. Snapshot, Clone and restore operations don't involve data copying. Also, Exporting the snapshot to another cluster doesn't have an impact on the Region Servers.
   
   _Note: Ensure, no schema change in a destination table like delete/rename of column family, table name, and etc._ 
   
   ### Steps : On Source Cluster

1. Create a table in the source cluster.
```
su - hbase

$ /usr/hdp/<hdp-version>/phoenix/bin/sqlline.py <ZK host>:2181:/<znode>

Example:
$ /usr/hdp/2.5.3.0-37/phoenix/bin/sqlline.py localhost:2181:/hbase-unsecure
```

```
CREATE TABLE IF NOT EXISTS US_POP_SAMPLE (
      state CHAR(2) NOT NULL,
      city VARCHAR NOT NULL,
      population BIGINT
      CONSTRAINT my_pk PRIMARY KEY (state, city));
```

2. Now let’s create a us_pop.csv file containing some data.

```
NY,New York,8143197
CA,Los Angeles,3844829
IL,Chicago,2842518
TX,Houston,2016582
```

3. Load data into the table using CsvImport tool.

```
su - hdfs

$ java -Dhdp.version=<version> -cp `hbase classpath`:  org.apache.phoenix.mapreduce.CsvBulkLoadTool   --table <US_POP_SAMPLE> --input /var/tmp/us_pop.csv
```

4. Create snapshot: You can take a snapshot of a table regardless of whether it is enabled or disabled. The snapshot operation doesn’t involve any data copying.

```
#su - hbase

$hbase shell

hbase> snapshot '<tablename>', '<snapshot-name>'

Example:
hbase> snapshot 'US_POP_SAMPLE', 'US_POP_SAMPLE_SNAPSHOT'
```

5. Now, the snapshot is ready. List all snapshots taken (by printing the names and relative information).
```
hbase> list_snapshots
```

6. Copy/export the snapshots to destination cluster.

   The ExportSnapshot tool copies all the data related to a snapshot (hfiles, logs, snapshot metadata) to the destination cluster.  tool executes a Map-Reduce job, similar to distcp, to copy files between the two clusters. and it works at file-system level the hbase cluster does not have to be online.
   
```
Example:
To copy a snapshot called US_POP_SAMPLE_SNAPSHOT to an HBase cluster srv2 (hdfs:///srv2:8082/hbase) using 16 mappers:

su - hdfs

$hbase org.apache.hadoop.hbase.snapshot.ExportSnapshot -snapshot US_POP_SAMPLE_SNAPSHOT -copy-to hdfs://srv2:8082/hbase -mappers 16
```

**or**  

You can take a backup to the localfilesystem.

a) First, Let's export to the local/source HDFS.

```
$hbase org.apache.hadoop.hbase.snapshot.ExportSnapshot -snapshot US_POP_SAMPLE_SNAPSHOT -copy-to <HDFS Path> -mappers 16

example:
$hbase org.apache.hadoop.hbase.snapshot.ExportSnapshot -snapshot us_pop_snapshot -copy-to /tmp/US_POP_SAMPLE_SNAPSHOT -mappers 16
```

b)  Now, copy the snapshot files from HDFS to LFS.

```
su - hdfs

$hdfs dfs -copyToLocal /tmp/US_POP_SAMPLE_SNAPSHOT /var/tmp/
```

### Steps: On Destination Cluster:

7. Create table schema as same as the source cluster.

```
su - hbase

$ /usr/hdp/<hdp-version>/phoenix/bin/sqlline.py <ZK host>:2181:/<znode>
```

```
CREATE TABLE IF NOT EXISTS US_POP_SAMPLE(
      state CHAR(2) NOT NULL,
      city VARCHAR NOT NULL,
      population BIGINT
      CONSTRAINT my_pk PRIMARY KEY (state, city));
 ```
 
 8. **Restore table**, The restore operation requires the table to be disabled, and the table will be restored to the state at the time when the snapshot was taken, changing both data and schema if required. 
 
 a) Disable table:
 
 ```
#su - hbase

$hbase shell
 
hbase> disable '<tablename>'

Example:
hbase> disable 'US_POP_SAMPLE'
```

b) Restore table from snapshot.

```
hbase> restore_snapshot '<snapshot-name>'
Example:
hbase> restore_snapshot 'US_POP_SAMPLE_SNAPSHOT'
```

c) Enable table after the snapshot restore.

```
hbase> enable '<tablename>'
Example:
hbase> enable 'US_POP_SAMPLE'
```

d) Done.

```
Example:
hbase>scan 'US_POP_SAMPLE'
```


_Note: Please copy the snapshot into appropriate HDFS location if you load it from LFS._
```
example :
$hdfs dfs -copyFromLocal /var/tmp/US_POP_SAMPLE_SNAPSHOT hdfs://<NN hostname>:8020/hbase
```


--------- 


**Other approach like CSV import/export**

https://phoenix.apache.org/hive_storage_handler.html

https://phoenix.apache.org/pig_integration.html

https://phoenix.apache.org/phoenix_spark.html

