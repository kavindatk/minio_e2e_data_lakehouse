# MINIO Cluster with Hive Metastore & Beeline Access 
## A Containerized, Enterprise-Ready On-Premises Data Lakehouse

<br/><br/>
<p align="center">
<picture>
  <img alt="docker" src="https://github.com/kavindatk/minio_e2e_data_lakehouse/blob/main/images/minio_logo.JPG" width="" height="125">
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


<br/>

2. Set up the Hive Metastore


3. Set up Hive Beeline


4. Testing Phase
