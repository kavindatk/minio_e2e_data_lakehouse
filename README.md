# MINIO Cluster with Hive Metastore & Beeline Access 
## A Containerized, Enterprise-Ready On-Premises Data Lakehouse

<br/><br/>
<p align="center">
<picture>
  <img alt="docker" src="https://github.com/kavindatk/minio_e2e_data_lakehouse/blob/main/images/minio_logo.JPG" width="" height="125">
</picture>
<picture>
  <img alt="docker" src="https://github.com/kavindatk/minio_e2e_data_lakehouse/blob/main/images/apache%20hive.JPG" width="" height="125">
</picture>  
</p>

### 🧩 Pre-Configuration Setup

<br/>
Before proceeding with the MINIO cluster setup, I performed a few OS-level configurations on all nodes to prepare the environment:

1. Defined the host list in the ```/etc/hosts``` file to ensure all nodes can communicate with each other by hostname instead of IP.
2. Set unique hostnames for each node ```(e.g., node1, node2, etc.).```
3. Verified network connectivity between all nodes using ping and ssh commands.
4. I have already installed HAProxy on VM4, which will be used for load balancing and access management.

<br/>

```bash
hadoop@node01:~$ hostname
node01
hadoop@node01:~$
hadoop@node01:~$ cat /etc/hosts
127.0.0.1 localhost
127.0.1.1 ubuntu-templ

# The following lines are desirable for IPv6 capable hosts
::1     ip6-localhost ip6-loopback
fe00::0 ip6-localnet
ff00::0 ip6-mcastprefix
ff02::1 ip6-allnodes
ff02::2 ip6-allrouters


172.27.16.193 node01
172.27.16.194 node02
172.27.16.195 node03
172.27.16.196 node04

172.27.16.196 minoproxy
hadoop@node01:~$

```

<br/>
<br/>

## ⚙️ Step 01 – Set up the MinIO Cluster

<p align="center">
<picture>
  <img alt="docker" src="https://github.com/kavindatk/minio_e2e_data_lakehouse/blob/main/images/minio_logo.JPG" width="" height="125">
</picture> 
</p>

In this step, I will show how to set up a MINIO cluster using Docker.
For this setup, I’m using the Docker network in host mode.

<br/>

I have created a data storage folder under the /mnt path, named data.
MINIO will use this folder to store the bucket (object storage) data.

<br/>

```bash
hadoop@node01:~$ cd /mnt/
hadoop@node01:/mnt$ sudo mkdir data
hadoop@node01:/mnt$ sudo chmod 777 data
hadoop@node01:/mnt$ ls -ls
total 4
4 drwxrwxrwx 2 root root 4096 Oct 22 16:08 data
hadoop@node01:/mnt$

```
<br/>
<br/>

You need to run the following Docker command on all four VMs to start the cluster.
Once all containers are running, you can verify the setup using the test command shown below.

<br/>

```bash
docker run -d --name minio_storage \
  --restart=always \
  --network host \
  -e MINIO_ROOT_USER='minioadmin' \
  -e MINIO_ROOT_PASSWORD='minioadmin' \
  -e MINIO_SERVER_ADDRESS=":9000" \
  -e MINIO_CONSOLE_ADDRESS=":9001" \
  -v /mnt/data1:/data \
  minio/minio:latest server \
    http://node01/data \
    http://node02/data \
    http://node03/data \
    http://node04/data 
```

```bash
# Can verify docker containner by using following commands
hadoop@node01:/opt/hive_config$ docker ps
CONTAINER ID   IMAGE                COMMAND                  CREATED         STATUS         PORTS     NAMES
2b5f7b746ca1   minio/minio:latest   "/usr/bin/docker-ent…"   2 minutes ago   Up 2 minutes             minio_storage
hadoop@node01:/opt/hive_config$
```

```bash
# Check the MINIO cluster status using mc client (MINIO client)

hadoop@node02:/mnt$ cd /home/hadoop/
hadoop@node02:~$
hadoop@node02:~$ wget https://dl.min.io/client/mc/release/linux-amd64/mc
--2025-10-22 16:23:21--  https://dl.min.io/client/mc/release/linux-amd64/mc
Resolving dl.min.io (dl.min.io)... 138.68.11.125, 178.128.69.202
Connecting to dl.min.io (dl.min.io)|138.68.11.125|:443... connected.
HTTP request sent, awaiting response... 200 OK
Length: 30535864 (29M) [application/octet-stream]
Saving to: ‘mc’

mc                                    100%[======================================================================>]  29.12M  6.64MB/s    in 4.4s

2025-10-22 16:23:26 (6.64 MB/s) - ‘mc’ saved [30535864/30535864]

hadoop@node02:~$ chmod +x mc
hadoop@node02:~$ sudo mv mc /usr/local/bin/
hadoop@node02:~$ mc --version
mc version RELEASE.2025-08-13T08-35-41Z (commit-id=7394ce0dd2a80935aded936b09fa12cbb3cb8096)
Runtime: go1.24.6 linux/amd64
Copyright (c) 2015-2025 MinIO, Inc.
License GNU AGPLv3 <https://www.gnu.org/licenses/agpl-3.0.html>
hadoop@node02:~$
hadoop@node02:~$

# Define alias

hadoop@node02:~$
hadoop@node02:~$ mc alias set myminio http://node01:9000 minioadmin minioadmin
``
mc: Configuration written to `/home/hadoop/.mc/config.json`. Please update your access credentials.
mc: Successfully created `/home/hadoop/.mc/share`.
mc: Initialized share uploads `/home/hadoop/.mc/share/uploads.json` file.
mc: Initialized share downloads `/home/hadoop/.mc/share/downloads.json` file.
Added `myminio` successfully.
hadoop@node02:~$


# Check the cluster health

hadoop@node02:~$
hadoop@node02:~$ mc admin info myminio
●  node01:9000
   Uptime: 6 minutes
   Version: 2025-09-07T16:13:09Z
   Network: 4/4 OK
   Drives: 1/1 OK
   Pool: 1

●  node02:9000
   Uptime: 8 minutes
   Version: 2025-09-07T16:13:09Z
   Network: 4/4 OK
   Drives: 1/1 OK
   Pool: 1

●  node03:9000
   Uptime: 8 minutes
   Version: 2025-09-07T16:13:09Z
   Network: 4/4 OK
   Drives: 1/1 OK
   Pool: 1

●  node04:9000
   Uptime: 8 minutes
   Version: 2025-09-07T16:13:09Z
   Network: 4/4 OK
   Drives: 1/1 OK
   Pool: 1

┌──────┬───────────────────────┬─────────────────────┬──────────────┐
│ Pool │ Drives Usage          │ Erasure stripe size │ Erasure sets │
│ 1st  │ 3.7% (total: 849 GiB) │ 4                   │ 1            │
└──────┴───────────────────────┴─────────────────────┴──────────────┘

4 drives online, 0 drives offline, EC:2
hadoop@node02:~$

```

<picture>
  <img alt="docker" src="https://github.com/kavindatk/minio_e2e_data_lakehouse/blob/main/images/minio%20cluster.JPG" width="600" height="400">
</picture>
<br/><br/>
<picture>
  <img alt="docker" src="https://github.com/kavindatk/minio_e2e_data_lakehouse/blob/main/images/minio%20web1.JPG" width="600" height="400">
</picture>
<br/><br/>
<picture>
  <img alt="docker" src="https://github.com/kavindatk/minio_e2e_data_lakehouse/blob/main/images/minio%20web2.JPG" width="600" height="400">
</picture>
<br/><br/>

once you created a bucket vis using web or terminal , it will be apper in the ```/mnt/data``` directory
<br/><br/>
### 💡 Important Note – Using HAProxy with MINIO

The most important part here is that I used HAProxy to access the MINIO cluster through a single interface.
In this setup, HAProxy handles the routing and directs requests to the available MINIO web UI automatically.

Below are some useful commands and the HAProxy configuration, which will help you understand how this setup works.

```bash
# Create alias for MINIO connection
mc alias set <alias name> http://<minio ip>:9000 <user> <password>
mc alias set myminio http://minio:9000 minioadmin minioadmin

# Create MINIO (S3) bucket 
mc mb <alias name>/<bucket name>
mc mb myminio/warehouse

# Enable buckt to public access (This need for our Hive setup)
mc anonymous set public <alias name>/<bucket name>
mc anonymous set public myminio/warehouse

# Upload files to bucket
mc cp <local file> <alias name>/<bucket>/<folder (optional)>
mc cp dataset.csv myminio/databucket/elec_cars
```


<br/>

## 🧩 Step 02 – Set up the Hive Metastore
## 🧩 Step 03 – Set up Hive Beeline

<p align="center">
<picture>
  <img alt="docker" src="https://github.com/kavindatk/minio_e2e_data_lakehouse/blob/main/images/apache%20hive.JPG" width="" height="125">
</picture>  
</p>

For Step 2 and Step 3, I’m planning to use <b>Docker Compose.</b>
At the moment, I don’t have a Docker cluster — instead, I have <b>four separate VMs with Docker installed</b>.
Because of that, I can’t deploy MINIO using Docker Compose.

Also, my <b>Hive Metastore and Hive Beeline</b> will both run on <b>VM 1</b>.
Among these, the most important component is the <b>Hive Metastore</b>, since it will be used later by <b>Iceberg, Trino/Presto, and Hue.</b>
<b>Beeline</b> is only needed for testing and running Hive SQL commands.

The following steps show how to set up <b>Hive Metastore and Hive Beeline</b>.   
<br>
### 🧩 Required Files Before Proceeding
<br/>
Before proceeding, we need the following files.
In this setup, I’m using Hive 4.0.0, and the files listed below are compatible with this version:
<br/><br/>
1. <b>hive-site.xml</b> – Hive configuration file<br/>
2. <b>core-site.xml</b> – Hadoop core configuration file (used by Hive)<br/>
3. <b>postgresql-42.7.3.jar</b> – Required for connecting Hive to PostgreSQL via JDBC<br/>
4. <b>hadoop-aws-3.3.4.jar</b> – Required for connecting Hive to MinIO<br/>
5. <b>aws-java-sdk-bundle-1.12.367.jar</b> – Required for connecting Hive to MinIO<br/>
<br/>
The <b>Hive 4.0.0 Docker image</b> already includes a basic Hadoop setup with the necessary libraries,
so I haven’t made any changes to the Hadoop configuration since it’s not part of my current requirement.
<br/>
I’ve uploaded the fully completed <b>XML and Docker Compose</b> files to my repository —
you can review them and adjust the configurations as needed.<br/>
Below, I’ll show only the <b>required commands</b> to run the setup.<br/>

```bash
# Folder structure

/opt/hive
├── conf
│   ├── core-site.xml
│   └── hive-site.xml
├── docker-compose.yml
├── hive_home
└── jars
    ├── aws-java-sdk-bundle-1.12.367.jar
    ├── hadoop-aws-3.3.4.jar
    └── postgresql-42.7.3.jar

```

```bash
# Start Docker container using Docker Compose

hadoop@node01:/opt/hive$
hadoop@node01:/opt/hive$ docker compose up -d
WARN[0000] /opt/hive/docker-compose.yml: the attribute `version` is obsolete, it will be ignored, please remove it to avoid potential confusion
[+] Running 5/5
 ✔ Network hive_minio_net       Created                                                                                                                0.1s
 ✔ Volume "hive_postgres_data"  Created                                                                                                                0.0s
 ✔ Container postgres_hive      Healthy                                                                                                               10.8s
 ✔ Container hive_server2       Started                                                                                                               11.1s
 ✔ Container hive_metastore     Started                                                                                                               11.0s
hadoop@node01:/opt/hive$
hadoop@node01:/opt/hive$

# Checking Dokcer status

hadoop@node01:/opt/hive$
hadoop@node01:/opt/hive$
hadoop@node01:/opt/hive$ docker ps
CONTAINER ID   IMAGE                COMMAND                  CREATED          STATUS                            PORTS                                                                                                        NAMES
cb46549aca57   apache/hive:4.0.0    "sh -c /entrypoint.sh"   16 seconds ago   Up 5 seconds (health: starting)   0.0.0.0:10000->10000/tcp, [::]:10000->10000/tcp, 9083/tcp, 0.0.0.0:10002->10002/tcp, [::]:10002->10002/tcp   hive_server2
2fffdcea04e9   apache/hive:4.0.0    "sh -c /entrypoint.sh"   16 seconds ago   Up 5 seconds (health: starting)   10000/tcp, 0.0.0.0:9083->9083/tcp, [::]:9083->9083/tcp, 10002/tcp                                            hive_metastore
3f45b80d4a72   postgres:13          "docker-entrypoint.s…"   16 seconds ago   Up 16 seconds (healthy)           0.0.0.0:5432->5432/tcp, [::]:5432->5432/tcp                                                                  postgres_hive
2b5f7b746ca1   minio/minio:latest   "/usr/bin/docker-ent…"   11 hours ago     Up 11 hours                                                                                                                                    minio_storage
hadoop@node01:/opt/hive$

```

<br/><br/>


## 🧪 Step 04 - Testing Phase
<br/>
Now it’s time for testing.
The following steps show the testing phase of the cluster. In this stage, I’ll demonstrate how to:
<br/><br/>
1. Create managed and external tables<br/>
2. Upload files and view data using Beeline<br/>
3. Insert data into tables using Beeline<br/>
<br/><br/>
Before we begin, there are a few initial configurations required — specifically, creating S3 buckets and setting the correct permissions.
The steps below show the complete process for these configurations.
<br/>


```bash
# Create warehouse bucket for store Hive data

hadoop@node01:/opt/hive/conf$ mc mb myminio/warehouse
Bucket created successfully `myminio/warehouse`.
hadoop@node01:/opt/hive/conf$ mc anonymous set public myminio/warehouse
Access permission for `myminio/warehouse` is set to `public`
hadoop@node01:/opt/hive/conf$

# Create tmpbucket bucket for store Hive tmp data

hadoop@node01:/opt/hive/conf$ mc mb myminio/tmpbucket
Bucket created successfully `myminio/tmpbucket`.
hadoop@node01:/opt/hive/conf$ mc anonymous set public myminio/tmpbucket
Access permission for `myminio/tmpbucket` is set to `public`
hadoop@node01:/opt/hive/conf$

```
<br/><br/>
### Beeline Testing
<br/><br/>
```bash
# Login docker containner using terminal and access the beeline
# docker exec -it hive_server2 beeline -u "jdbc:hive2:///"  <- this can use for direct login 

hadoop@node01:/opt/hive$
hadoop@node01:/opt/hive$ docker exec -it hive_server2 bash
hive@79777dae2ffa:/opt/hive$ beeline -u "jdbc:hive2:///"
SLF4J: Class path contains multiple SLF4J bindings.
SLF4J: Found binding in [jar:file:/opt/hive/lib/log4j-slf4j-impl-2.18.0.jar!/org/slf4j/impl/StaticLoggerBinder.class]
SLF4J: Found binding in [jar:file:/opt/hadoop/share/hadoop/common/lib/slf4j-reload4j-1.7.36.jar!/org/slf4j/impl/StaticLoggerBinder.class]
SLF4J: See http://www.slf4j.org/codes.html#multiple_bindings for an explanation.
SLF4J: Actual binding is of type [org.apache.logging.slf4j.Log4jLoggerFactory]
SLF4J: Class path contains multiple SLF4J bindings.
SLF4J: Found binding in [jar:file:/opt/hive/lib/log4j-slf4j-impl-2.18.0.jar!/org/slf4j/impl/StaticLoggerBinder.class]
SLF4J: Found binding in [jar:file:/opt/hadoop/share/hadoop/common/lib/slf4j-reload4j-1.7.36.jar!/org/slf4j/impl/StaticLoggerBinder.class]
SLF4J: See http://www.slf4j.org/codes.html#multiple_bindings for an explanation.
SLF4J: Actual binding is of type [org.apache.logging.slf4j.Log4jLoggerFactory]
Connecting to jdbc:hive2:///
25/10/23 06:44:16 [main]: WARN conf.HiveConf: HiveConf of name hive.default.table.type does not exist
Hive Session ID = 24ea6554-0b27-4e50-a972-8b7ce8bdaef3
25/10/23 06:44:20 [main]: WARN exec.FunctionRegistry: UDF Class org.apache.hive.org.apache.datasketches.hive.cpc.UnionSketchUDF does not have description. Please annotate the class with the org.apache.hadoop.hive.ql.exec.Description annotation and provide the description of the function.
25/10/23 06:44:20 [main]: WARN exec.FunctionRegistry: UDF Class org.apache.hive.org.apache.datasketches.hive.hll.UnionSketchUDF does not have description. Please annotate the class with the org.apache.hadoop.hive.ql.exec.Description annotation and provide the description of the function.
25/10/23 06:44:20 [main]: WARN exec.FunctionRegistry: UDF Class org.apache.hive.org.apache.datasketches.hive.theta.IntersectSketchUDF does not have description. Please annotate the class with the org.apache.hadoop.hive.ql.exec.Description annotation and provide the description of the function.
25/10/23 06:44:20 [main]: WARN exec.FunctionRegistry: UDF Class org.apache.hive.org.apache.datasketches.hive.theta.EstimateSketchUDF does not have description. Please annotate the class with the org.apache.hadoop.hive.ql.exec.Description annotation and provide the description of the function.
25/10/23 06:44:20 [main]: WARN exec.FunctionRegistry: UDF Class org.apache.hive.org.apache.datasketches.hive.theta.ExcludeSketchUDF does not have description. Please annotate the class with the org.apache.hadoop.hive.ql.exec.Description annotation and provide the description of the function.
25/10/23 06:44:20 [main]: WARN exec.FunctionRegistry: UDF Class org.apache.hive.org.apache.datasketches.hive.theta.UnionSketchUDF does not have description. Please annotate the class with the org.apache.hadoop.hive.ql.exec.Description annotation and provide the description of the function.
25/10/23 06:44:20 [main]: WARN exec.FunctionRegistry: UDF Class org.apache.hive.org.apache.datasketches.hive.tuple.ArrayOfDoublesSketchToValuesUDTF does not have description. Please annotate the class with the org.apache.hadoop.hive.ql.exec.Description annotation and provide the description of the function.
25/10/23 06:44:21 [main]: WARN session.SessionState: Configuration hive.reloadable.aux.jars.path not specified
Connected to: Apache Hive (version 4.0.0)
Driver: Hive JDBC (version 4.0.0)
Transaction isolation: TRANSACTION_REPEATABLE_READ
Beeline version 4.0.0 by Apache Hive
0: jdbc:hive2:///> show databases;
25/10/23 06:44:26 [Metastore-RuntimeStats-Loader-1]: WARN conf.HiveConf: HiveConf of name hive.default.table.type does not exist
+----------------+
| database_name  |
+----------------+
| default        |
+----------------+
1 row selected (2.385 seconds)
0: jdbc:hive2:///>

```
<br/><br/>


```bash
# Create some database/tables and load data to tables 

0: jdbc:hive2:///>
0: jdbc:hive2:///>
0: jdbc:hive2:///> show databases;
25/10/23 06:48:14 [Metastore-RuntimeStats-Loader-1]: WARN conf.HiveConf: HiveConf of name hive.default.table.type does not exist
+----------------+
| database_name  |
+----------------+
| default        |
+----------------+
1 row selected (2.375 seconds)
0: jdbc:hive2:///> create database minio_test;
No rows affected (2.506 seconds)
0: jdbc:hive2:///>
0: jdbc:hive2:///> show databases;
+----------------+
| database_name  |
+----------------+
| default        |
| minio_test     |
+----------------+
2 rows selected (0.046 seconds)
0: jdbc:hive2:///>
0: jdbc:hive2:///> use minio_test;
No rows affected (0.093 seconds)
0: jdbc:hive2:///>
0: jdbc:hive2:///> CREATE TABLE cars_managed_table (
. . . . . . . . .>   id INT,
. . . . . . . . .>   make STRING,
. . . . . . . . .>   model STRING,
. . . . . . . . .>   year INT
. . . . . . . . .> );
No rows affected (1.02 seconds)
0: jdbc:hive2:///>
0: jdbc:hive2:///> INSERT INTO cars_managed_table VALUES
. . . . . . . . .>   (1, 'Tesla', 'Model S', 2022),
. . . . . . . . .>   (2, 'Nissan', 'Leaf', 2021),
. . . . . . . . .>   (3, 'Chevrolet', 'Bolt', 2020);
25/10/23 06:50:00 [HiveServer2-Background-Pool: Thread-58]: WARN ql.Driver: Hive-on-MR is deprecated in Hive 2 and may not be available in the future versions. Consider using a different execution engine (i.e. tez) or using Hive 1.X releases.
Query ID = hive_20251023064957_299777a0-e0e1-400e-97c7-5a6473776b44
Total jobs = 3
Launching Job 1 out of 3
Number of reduce tasks determined at compile time: 1
In order to change the average load for a reducer (in bytes):
  set hive.exec.reducers.bytes.per.reducer=<number>
In order to limit the maximum number of reducers:
  set hive.exec.reducers.max=<number>
In order to set a constant number of reducers:
  set mapreduce.job.reduces=<number>
25/10/23 06:50:00 [HiveServer2-Background-Pool: Thread-58]: WARN impl.MetricsSystemImpl: JobTracker metrics system already initialized!
25/10/23 06:50:00 [HiveServer2-Background-Pool: Thread-58]: WARN mapreduce.JobResourceUploader: Hadoop command-line option parsing not performed. Implement the Tool interface and execute your application with ToolRunner to remedy this.
WARN  : Hive-on-MR is deprecated in Hive 2 and may not be available in the future versions. Consider using a different execution engine (i.e. tez) or using Hive 1.X releases.
Job running in-process (local Hadoop)
25/10/23 06:50:01 [LocalJobRunner Map Task Executor #0]: WARN objectinspector.StandardStructObjectInspector: Trying to access 3 fields inside a list of 2 elements: [3, 20.0]
25/10/23 06:50:01 [LocalJobRunner Map Task Executor #0]: WARN objectinspector.StandardStructObjectInspector: ignoring similar errors.
25/10/23 06:50:01 [pool-9-thread-1]: WARN impl.MetricsSystemImpl: JobTracker metrics system already initialized!
2025-10-23 06:50:01,963 Stage-1 map = 100%,  reduce = 100%, Cumulative CPU 1.41 sec
MapReduce Total cumulative CPU time: 1 seconds 410 msec
Ended Job = job_local294063186_0001
Stage-4 is selected by condition resolver.
Stage-3 is filtered out by condition resolver.
Stage-5 is filtered out by condition resolver.
Loading data to table minio_test.cars_managed_table
MapReduce Jobs Launched:
Stage-Stage-1:  Cumulative CPU: 1.41 sec   HDFS Read: 0 HDFS Write: 0 SUCCESS
Total MapReduce CPU Time Spent: 1 seconds 410 msec
3 rows affected (5.51 seconds)
0: jdbc:hive2:///>
0: jdbc:hive2:///> show tables;
+---------------------+
|      tab_name       |
+---------------------+
| cars_managed_table  |
+---------------------+
1 row selected (0.116 seconds)
0: jdbc:hive2:///>
0: jdbc:hive2:///> select * from cars_managed_table;
+------------------------+--------------------------+---------------------------+--------------------------+
| cars_managed_table.id  | cars_managed_table.make  | cars_managed_table.model  | cars_managed_table.year  |
+------------------------+--------------------------+---------------------------+--------------------------+
| 1                      | Tesla                    | Model S                   | 2022                     |
| 2                      | Nissan                   | Leaf                      | 2021                     |
| 3                      | Chevrolet                | Bolt                      | 2020                     |
+------------------------+--------------------------+---------------------------+--------------------------+
3 rows selected (0.631 seconds)
0: jdbc:hive2:///>
0: jdbc:hive2:///>
0: jdbc:hive2:///>
0: jdbc:hive2:///>
0: jdbc:hive2:///> CREATE EXTERNAL TABLE cars_external_table (
. . . . . . . . .>   id INT,
. . . . . . . . .>   make STRING,
. . . . . . . . .>   model STRING,
. . . . . . . . .>   year INT
. . . . . . . . .> );
No rows affected (0.192 seconds)
0: jdbc:hive2:///>
0: jdbc:hive2:///> INSERT INTO cars_external_table VALUES
. . . . . . . . .>   (1, 'Tesla', 'Model S', 2022),
. . . . . . . . .>   (2, 'Nissan', 'Leaf', 2021),
. . . . . . . . .>   (3, 'Chevrolet', 'Bolt', 2020);
25/10/23 06:51:11 [HiveServer2-Background-Pool: Thread-122]: WARN ql.Driver: Hive-on-MR is deprecated in Hive 2 and may not be available in the future versions. Consider using a different execution engine (i.e. tez) or using Hive 1.X releases.
Query ID = hive_20251023065111_e47f81de-e315-4340-908b-a8da680b5c57
Total jobs = 3
Launching Job 1 out of 3
Number of reduce tasks determined at compile time: 1
In order to change the average load for a reducer (in bytes):
  set hive.exec.reducers.bytes.per.reducer=<number>
In order to limit the maximum number of reducers:
  set hive.exec.reducers.max=<number>
In order to set a constant number of reducers:
  set mapreduce.job.reduces=<number>
25/10/23 06:51:11 [HiveServer2-Background-Pool: Thread-122]: WARN impl.MetricsSystemImpl: JobTracker metrics system already initialized!
25/10/23 06:51:12 [HiveServer2-Background-Pool: Thread-122]: WARN impl.MetricsSystemImpl: JobTracker metrics system already initialized!
25/10/23 06:51:12 [HiveServer2-Background-Pool: Thread-122]: WARN mapreduce.JobResourceUploader: Hadoop command-line option parsing not performed. Implement the Tool interface and execute your application with ToolRunner to remedy this.
Job running in-process (local Hadoop)
25/10/23 06:51:12 [pool-18-thread-1]: WARN impl.MetricsSystemImpl: JobTracker metrics system already initialized!
WARN  : Hive-on-MR is deprecated in Hive 2 and may not be available in the future versions. Consider using a different execution engine (i.e. tez) or using Hive 1.X releases.
2025-10-23 06:51:13,431 Stage-1 map = 100%,  reduce = 100%, Cumulative CPU 0.67 sec
MapReduce Total cumulative CPU time: 670 msec
Ended Job = job_local1108689548_0002
Stage-4 is selected by condition resolver.
Stage-3 is filtered out by condition resolver.
Stage-5 is filtered out by condition resolver.
Loading data to table minio_test.cars_external_table
25/10/23 06:51:13 [HiveServer2-Background-Pool: Thread-122]: WARN metadata.Hive: Cannot get a table snapshot for cars_external_table
25/10/23 06:51:14 [HiveServer2-Background-Pool: Thread-122]: WARN metadata.Hive: Cannot get a table snapshot for cars_external_table
MapReduce Jobs Launched:
Stage-Stage-1:  Cumulative CPU: 0.67 sec   HDFS Read: 0 HDFS Write: 0 SUCCESS
Total MapReduce CPU Time Spent: 670 msec
3 rows affected (2.515 seconds)
0: jdbc:hive2:///>
0: jdbc:hive2:///> show tables;
+----------------------+
|       tab_name       |
+----------------------+
| cars_external_table  |
| cars_managed_table   |
+----------------------+
2 rows selected (0.039 seconds)
0: jdbc:hive2:///>
0: jdbc:hive2:///> select * from cars_external_table;
25/10/23 06:51:24 [5fd1b7bd-0662-4dd7-8eff-da012ad7c023 main]: WARN optimizer.SimpleFetchOptimizer: Table minio_test@cars_external_table is external table, falling back to filesystem scan.
+-------------------------+---------------------------+----------------------------+---------------------------+
| cars_external_table.id  | cars_external_table.make  | cars_external_table.model  | cars_external_table.year  |
+-------------------------+---------------------------+----------------------------+---------------------------+
| 1                       | Tesla                     | Model S                    | 2022                      |
| 2                       | Nissan                    | Leaf                       | 2021                      |
| 3                       | Chevrolet                 | Bolt                       | 2020                      |
+-------------------------+---------------------------+----------------------------+---------------------------+
3 rows selected (0.326 seconds)
0: jdbc:hive2:///>
0: jdbc:hive2:///>
0: jdbc:hive2:///>
0: jdbc:hive2:///> describe formatted cars_external_table;
+-------------------------------+----------------------------------------------------+----------------------------------------------------+
|           col_name            |                     data_type                      |                      comment                       |
+-------------------------------+----------------------------------------------------+----------------------------------------------------+
| id                            | int                                                |                                                    |
| make                          | string                                             |                                                    |
| model                         | string                                             |                                                    |
| year                          | int                                                |                                                    |
|                               | NULL                                               | NULL                                               |
| # Detailed Table Information  | NULL                                               | NULL                                               |
| Database:                     | minio_test                                         | NULL                                               |
| OwnerType:                    | USER                                               | NULL                                               |
| Owner:                        | hive                                               | NULL                                               |
| CreateTime:                   | Thu Oct 23 06:51:04 UTC 2025                       | NULL                                               |
| LastAccessTime:               | UNKNOWN                                            | NULL                                               |
| Retention:                    | 0                                                  | NULL                                               |
| Location:                     | s3a://warehouse/tablespace/external_tables/minio_test.db/cars_external_table | NULL                                               |
| Table Type:                   | EXTERNAL_TABLE                                     | NULL                                               |
| Table Parameters:             | NULL                                               | NULL                                               |
|                               | COLUMN_STATS_ACCURATE                              | {\"BASIC_STATS\":\"true\",\"COLUMN_STATS\":{\"id\":\"true\",\"make\":\"true\",\"model\":\"true\",\"year\":\"true\"}} |
|                               | EXTERNAL                                           | TRUE                                               |
|                               | bucketing_version                                  | 2                                                  |
|                               | numFiles                                           | 1                                                  |
|                               | numRows                                            | 3                                                  |
|                               | rawDataSize                                        | 59                                                 |
|                               | totalSize                                          | 62                                                 |
|                               | transient_lastDdlTime                              | 1761202274                                         |
|                               | NULL                                               | NULL                                               |
| # Storage Information         | NULL                                               | NULL                                               |
| SerDe Library:                | org.apache.hadoop.hive.serde2.lazy.LazySimpleSerDe | NULL                                               |
| InputFormat:                  | org.apache.hadoop.mapred.TextInputFormat           | NULL                                               |
| OutputFormat:                 | org.apache.hadoop.hive.ql.io.HiveIgnoreKeyTextOutputFormat | NULL                                               |
| Compressed:                   | No                                                 | NULL                                               |
| Num Buckets:                  | -1                                                 | NULL                                               |
| Bucket Columns:               | []                                                 | NULL                                               |
| Sort Columns:                 | []                                                 | NULL                                               |
| Storage Desc Params:          | NULL                                               | NULL                                               |
|                               | serialization.format                               | 1                                                  |
+-------------------------------+----------------------------------------------------+----------------------------------------------------+
34 rows selected (0.122 seconds)
0: jdbc:hive2:///>
0: jdbc:hive2:///> describe formatted cars_managed_table;
+-------------------------------+----------------------------------------------------+----------------------------------------------------+
|           col_name            |                     data_type                      |                      comment                       |
+-------------------------------+----------------------------------------------------+----------------------------------------------------+
| id                            | int                                                |                                                    |
| make                          | string                                             |                                                    |
| model                         | string                                             |                                                    |
| year                          | int                                                |                                                    |
|                               | NULL                                               | NULL                                               |
| # Detailed Table Information  | NULL                                               | NULL                                               |
| Database:                     | minio_test                                         | NULL                                               |
| OwnerType:                    | USER                                               | NULL                                               |
| Owner:                        | hive                                               | NULL                                               |
| CreateTime:                   | Thu Oct 23 06:49:50 UTC 2025                       | NULL                                               |
| LastAccessTime:               | UNKNOWN                                            | NULL                                               |
| Retention:                    | 0                                                  | NULL                                               |
| Location:                     | s3a://warehouse/tablespace/managed_tables/minio_test.db/cars_managed_table | NULL                                               |
| Table Type:                   | MANAGED_TABLE                                      | NULL                                               |
| Table Parameters:             | NULL                                               | NULL                                               |
|                               | COLUMN_STATS_ACCURATE                              | {\"BASIC_STATS\":\"true\",\"COLUMN_STATS\":{\"id\":\"true\",\"make\":\"true\",\"model\":\"true\",\"year\":\"true\"}} |
|                               | bucketing_version                                  | 2                                                  |
|                               | numFiles                                           | 1                                                  |
|                               | numRows                                            | 3                                                  |
|                               | rawDataSize                                        | 59                                                 |
|                               | totalSize                                          | 62                                                 |
|                               | transactional                                      | true                                               |
|                               | transactional_properties                           | insert_only                                        |
|                               | transient_lastDdlTime                              | 1761202202                                         |
|                               | NULL                                               | NULL                                               |
| # Storage Information         | NULL                                               | NULL                                               |
| SerDe Library:                | org.apache.hadoop.hive.serde2.lazy.LazySimpleSerDe | NULL                                               |
| InputFormat:                  | org.apache.hadoop.mapred.TextInputFormat           | NULL                                               |
| OutputFormat:                 | org.apache.hadoop.hive.ql.io.HiveIgnoreKeyTextOutputFormat | NULL                                               |
| Compressed:                   | No                                                 | NULL                                               |
| Num Buckets:                  | -1                                                 | NULL                                               |
| Bucket Columns:               | []                                                 | NULL                                               |
| Sort Columns:                 | []                                                 | NULL                                               |
| Storage Desc Params:          | NULL                                               | NULL                                               |
|                               | serialization.format                               | 1                                                  |
+-------------------------------+----------------------------------------------------+----------------------------------------------------+
35 rows selected (0.104 seconds)
0: jdbc:hive2:///>

```
<br/><br/>

<picture>
  <img alt="docker" src="https://github.com/kavindatk/minio_e2e_data_lakehouse/blob/main/images/minio%20web3.JPG" width="" height="">
</picture>

<picture>
  <img alt="docker" src="https://github.com/kavindatk/minio_e2e_data_lakehouse/blob/main/images/minio%20web4.JPG" width="" height="">
</picture>

<br/><br/>

```bash
# I have created carsinfobucket S3 bucket and upload CSV file

hadoop@node01:/opt/hive_config$
hadoop@node01:/opt/hive_config$ mc mb myminio/carsinfobucket
Bucket created successfully `myminio/carsinfobucket`.
hadoop@node01:/opt/hive_config$ mc cp dataset.csv myminio/carsinfobucket/cars_details/
.../hive_config/dataset.csv: 61.31 MiB / 61.31 MiB ┃▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓┃ 62.67 MiB/s 0shadoop@node01:/opt/hive_config$
hadoop@node01:/opt/hive_config$ mc ls  myminio/carsinfobucket/cars_details
[2025-10-23 06:57:27 UTC]  61MiB STANDARD cars_details
hadoop@node01:/opt/hive_config$
hadoop@node01:/opt/hive_config$


# Now , Lets create some table and load external data

0: jdbc:hive2:///>
0: jdbc:hive2:///>
0: jdbc:hive2:///> CREATE EXTERNAL TABLE IF NOT EXISTS cars_info_bucket_data (
. . . . . . . . .>   VIN STRING,
. . . . . . . . .>   County STRING,
. . . . . . . . .>   City STRING,
. . . . . . . . .>   State STRING,
. . . . . . . . .>   Postal_Code STRING,
. . . . . . . . .>   Model_Year INT,
. . . . . . . . .>   Make STRING,
. . . . . . . . .>   Model STRING,
. . . . . . . . .>   Electric_Vehicle_Type STRING,
. . . . . . . . .>   CAFV_Eligibility STRING,
. . . . . . . . .>   Electric_Range INT,
. . . . . . . . .>   Base_MSRP INT,
. . . . . . . . .>   Legislative_District STRING,
. . . . . . . . .>   DOL_Vehicle_ID STRING,
. . . . . . . . .>   Vehicle_Location STRING,
. . . . . . . . .>   Electric_Utility STRING,
. . . . . . . . .>   Census_Tract STRING
. . . . . . . . .> )
. . . . . . . . .> ROW FORMAT SERDE 'org.apache.hadoop.hive.serde2.OpenCSVSerde'
. . . . . . . . .> WITH SERDEPROPERTIES (
. . . . . . . . .>   "separatorChar" = ",",
. . . . . . . . .>   "quoteChar"     = "\"",
. . . . . . . . .>   "escapeChar"    = "\\"
. . . . . . . . .> )
. . . . . . . . .> STORED AS TEXTFILE
. . . . . . . . .> LOCATION 's3a://carsinfobucket/cars_details/'
. . . . . . . . .> TBLPROPERTIES ("skip.header.line.count"="1");
No rows affected (0.171 seconds)
0: jdbc:hive2:///>
0: jdbc:hive2:///> show tables;
+------------------------+
|        tab_name        |
+------------------------+
| cars_external_table    |
| cars_info_bucket_data  |
| cars_managed_table     |
+------------------------+
3 rows selected (0.031 seconds)
0: jdbc:hive2:///>
0: jdbc:hive2:///> select * from cars_info_bucket_data limit 10;
25/10/23 07:09:20 [710da8d9-92dd-4a05-82b4-4b31a101dce4 main]: WARN calcite.RelOptHiveTable: No Stats for minio_test@cars_info_bucket_data, Columns: legislative_district, electric_utility, city, county, model_year, vehicle_location, base_msrp, electric_vehicle_type, vin, model, dol_vehicle_id, state, postal_code, cafv_eligibility, make, census_tract, electric_range
No Stats for minio_test@cars_info_bucket_data, Columns: legislative_district, electric_utility, city, county, model_year, vehicle_location, base_msrp, electric_vehicle_type, vin, model, dol_vehicle_id, state, postal_code, cafv_eligibility, make, census_tract, electric_range
25/10/23 07:09:20 [710da8d9-92dd-4a05-82b4-4b31a101dce4 main]: WARN optimizer.SimpleFetchOptimizer: Table minio_test@cars_info_bucket_data is external table, falling back to filesystem scan.
+----------------------------+-------------------------------+-----------------------------+------------------------------+------------------------------------+-----------------------------------+-----------------------------+------------------------------+----------------------------------------------+----------------------------------------------------+---------------------------------------+----------------------------------+---------------------------------------------+---------------------------------------+-----------------------------------------+------------------------------------------------+-------------------------------------+
| cars_info_bucket_data.vin  | cars_info_bucket_data.county  | cars_info_bucket_data.city  | cars_info_bucket_data.state  | cars_info_bucket_data.postal_code  | cars_info_bucket_data.model_year  | cars_info_bucket_data.make  | cars_info_bucket_data.model  | cars_info_bucket_data.electric_vehicle_type  |       cars_info_bucket_data.cafv_eligibility       | cars_info_bucket_data.electric_range  | cars_info_bucket_data.base_msrp  | cars_info_bucket_data.legislative_district  | cars_info_bucket_data.dol_vehicle_id  | cars_info_bucket_data.vehicle_location  |     cars_info_bucket_data.electric_utility     | cars_info_bucket_data.census_tract  |
+----------------------------+-------------------------------+-----------------------------+------------------------------+------------------------------------+-----------------------------------+-----------------------------+------------------------------+----------------------------------------------+----------------------------------------------------+---------------------------------------+----------------------------------+---------------------------------------------+---------------------------------------+-----------------------------------------+------------------------------------------------+-------------------------------------+
| WA1E2AFY8R                 | Thurston                      | Olympia                     | WA                           | 98512                              | 2024                              | AUDI                        | Q5 E                         | Plug-in Hybrid Electric Vehicle (PHEV)       | Not eligible due to low battery range              | 23                                    | 0                                | 22                                          | 263239938                             | POINT (-122.90787 46.9461)              | PUGET SOUND ENERGY INC                         | 53067010910                         |
| WAUUPBFF4J                 | Yakima                        | Wapato                      | WA                           | 98951                              | 2018                              | AUDI                        | A3                           | Plug-in Hybrid Electric Vehicle (PHEV)       | Not eligible due to low battery range              | 16                                    | 0                                | 15                                          | 318160860                             | POINT (-120.42083 46.44779)             | PACIFICORP                                     | 53077940008                         |
| 1N4AZ0CP0F                 | King                          | Seattle                     | WA                           | 98125                              | 2015                              | NISSAN                      | LEAF                         | Battery Electric Vehicle (BEV)               | Clean Alternative Fuel Vehicle Eligible            | 84                                    | 0                                | 46                                          | 184963586                             | POINT (-122.30253 47.72656)             | CITY OF SEATTLE - (WA)|CITY OF TACOMA - (WA)   | 53033000700                         |
| WA1VAAGE5K                 | King                          | Kent                        | WA                           | 98031                              | 2019                              | AUDI                        | E-TRON                       | Battery Electric Vehicle (BEV)               | Clean Alternative Fuel Vehicle Eligible            | 204                                   | 0                                | 11                                          | 259426821                             | POINT (-122.17743 47.41185)             | PUGET SOUND ENERGY INC||CITY OF TACOMA - (WA)  | 53033029306                         |
| 7SAXCAE57N                 | Snohomish                     | Bothell                     | WA                           | 98021                              | 2022                              | TESLA                       | MODEL X                      | Battery Electric Vehicle (BEV)               | Eligibility unknown as battery range has not been researched | 0                                     | 0                                | 1                                           | 208182236                             | POINT (-122.18384 47.8031)              | PUGET SOUND ENERGY INC                         | 53061051922                         |
| KNDJP3AEXG                 | Snohomish                     | Lynnwood                    | WA                           | 98037                              | 2016                              | KIA                         | SOUL                         | Battery Electric Vehicle (BEV)               | Clean Alternative Fuel Vehicle Eligible            | 93                                    | 31950                            | 21                                          | 209171889                             | POINT (-122.27734 47.83785)             | PUGET SOUND ENERGY INC                         | 53061051928                         |
| 1N4AZ1CP7K                 | Snohomish                     | Edmonds                     | WA                           | 98026                              | 2019                              | NISSAN                      | LEAF                         | Battery Electric Vehicle (BEV)               | Clean Alternative Fuel Vehicle Eligible            | 150                                   | 0                                | 32                                          | 478448624                             | POINT (-122.31768 47.87166)             | PUGET SOUND ENERGY INC                         | 53061050900                         |
| KNDCC3LG4L                 | Snohomish                     | Brier                       | WA                           | 98036                              | 2020                              | KIA                         | NIRO                         | Battery Electric Vehicle (BEV)               | Clean Alternative Fuel Vehicle Eligible            | 239                                   | 0                                | 1                                           | 281365724                             | POINT (-122.29245 47.82557)             | PUGET SOUND ENERGY INC                         | 53061051914                         |
| KM8JFDD27S                 | Kitsap                        | Bremerton                   | WA                           | 98311                              | 2025                              | HYUNDAI                     | TUCSON                       | Plug-in Hybrid Electric Vehicle (PHEV)       | Clean Alternative Fuel Vehicle Eligible            | 32                                    | 0                                | 23                                          | 281377456                             | POINT (-122.60915 47.62631)             | PUGET SOUND ENERGY INC                         | 53035091600                         |
| 1C4JJXR6XM                 | Sauk                          | Spring Green                | WI                           | 53588                              | 2021                              | JEEP                        | WRANGLER                     | Plug-in Hybrid Electric Vehicle (PHEV)       | Not eligible due to low battery range              | 21                                    | 0                                |                                             | 182273605                             | POINT (-90.06403 43.17838)              | NON WASHINGTON STATE ELECTRIC UTILITY          | 55111000800                         |
+----------------------------+-------------------------------+-----------------------------+------------------------------+------------------------------------+-----------------------------------+-----------------------------+------------------------------+----------------------------------------------+----------------------------------------------------+---------------------------------------+----------------------------------+---------------------------------------------+---------------------------------------+-----------------------------------------+------------------------------------------------+-------------------------------------+
10 rows selected (0.48 seconds)
0: jdbc:hive2:///>
0: jdbc:hive2:///>

# Now you can run some queries like below

0: jdbc:hive2:///>
0: jdbc:hive2:///>
0: jdbc:hive2:///> select count(*) from cars_info_bucket_data;
25/10/23 07:10:56 [HiveServer2-Background-Pool: Thread-96]: WARN ql.Driver: Hive-on-MR is deprecated in Hive 2 and may not be available in the future versions. Consider using a different execution engine (i.e. tez) or using Hive 1.X releases.
Query ID = hive_20251023071056_8001af9c-2626-4808-a791-95ec9cf5b5fd
Total jobs = 1
Launching Job 1 out of 1
Number of reduce tasks determined at compile time: 1
In order to change the average load for a reducer (in bytes):
  set hive.exec.reducers.bytes.per.reducer=<number>
In order to limit the maximum number of reducers:
  set hive.exec.reducers.max=<number>
In order to set a constant number of reducers:
  set mapreduce.job.reduces=<number>
25/10/23 07:10:56 [HiveServer2-Background-Pool: Thread-96]: WARN impl.MetricsSystemImpl: JobTracker metrics system already initialized!
25/10/23 07:10:56 [HiveServer2-Background-Pool: Thread-96]: WARN mapreduce.JobResourceUploader: Hadoop command-line option parsing not performed. Implement the Tool interface and execute your application with ToolRunner to remedy this.
Job running in-process (local Hadoop)
WARN  : Hive-on-MR is deprecated in Hive 2 and may not be available in the future versions. Consider using a different execution engine (i.e. tez) or using Hive 1.X releases.
25/10/23 07:10:58 [HiveServer2-Background-Pool: Thread-96]: WARN mapreduce.Counters: Group org.apache.hadoop.mapred.Task$Counter is deprecated. Use org.apache.hadoop.mapreduce.TaskCounter instead
2025-10-23 07:10:58,576 Stage-1 map = 0%,  reduce = 0%
25/10/23 07:11:01 [pool-10-thread-1]: WARN impl.MetricsSystemImpl: JobTracker metrics system already initialized!
2025-10-23 07:11:02,629 Stage-1 map = 100%,  reduce = 100%, Cumulative CPU 5.85 sec
MapReduce Total cumulative CPU time: 5 seconds 850 msec
Ended Job = job_local18412293_0001
MapReduce Jobs Launched:
Stage-Stage-1:  Cumulative CPU: 5.85 sec   HDFS Read: 0 HDFS Write: 0 SUCCESS
Total MapReduce CPU Time Spent: 5 seconds 850 msec
+---------+
|   _c0   |
+---------+
| 264629  |
+---------+
1 row selected (6.828 seconds)
0: jdbc:hive2:///>

```
