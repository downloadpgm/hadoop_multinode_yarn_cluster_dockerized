# Hadoop running in YARN cluster in Docker

Apache Hadoop YARN is a resource management and job scheduling technology in the open source Hadoop distributed processing framework.

This Docker image contains Hadoop binaries prebuilt and uploaded in Docker Hub.

## Steps to Build Hadoop image
```shell
$ wget https://archive.apache.org/dist/hadoop/common/hadoop-2.7.3/hadoop-2.7.3.tar.gz
$ docker image build -t mkenjis/ubhdpclu_vol_img .
$ docker login   # provide user and password
$ docker image push mkenjis/ubhdpclu_vol_img
```

## Shell Scripts Inside 

> run_hadoop.sh

Sets up the environment for the YARN cluster by executing the following steps :
- sets environment variables for JAVA and HADOOP
- starts the SSH service and scans the slave nodes for passwordless SSH
- copies the Hadoop configuration files to slave nodes
- initializes the HDFS filesystem
- starts Hadoop Name node and Data nodes
- starts YARN Resource and Node managers

> create_conf_files.sh

Creates the following Hadoop files $HADOOP_HOME/etc/hadoop directory :
- core-site.xml
- hdfs-site.xml
- mapred-site.xml
- yarn-site.xml
- hadoop-env.sh

## Start Swarm cluster

1. start swarm mode in node1
```shell
$ docker swarm init --advertise-addr <IP node1>
$ docker swarm join-token worker  # issue a token to add a node as worker to swarm
```

2. add 3 more workers in swarm cluster (node2, node3, node4)
```shell
$ docker swarm join --token <token> <IP nodeN>:2377
```

3. label eacho node to anchor each container in swarm cluster
```shell
docker node update --label-add hostlabel=hdpmst node1
docker node update --label-add hostlabel=hdp1 node2
docker node update --label-add hostlabel=hdp2 node3
docker node update --label-add hostlabel=hdp3 node4
```

4. create an external "overlay" network in swarm to link the 2 stacks (hdp and spk)
```shell
docker network create --driver overlay mynet
```

5. start hadoop namenode and datanodes 
```shell
$ docker stack deploy -c docker-compose.yml hdp
$ docker service ls
ID             NAME             MODE         REPLICAS   IMAGE                             PORTS
t3s7ud9u21hr   hdp_hdpmst    replicated      1/1        mkenjis/ubhdpclu_vol_img:latest   
mi3w7xvf9vyt   hdp_hdp1      replicated      1/1        mkenjis/ubhdpclu_vol_img:latest   
xlg5ww9q0v6j   hdp_hdp2      replicated      1/1        mkenjis/ubhdpclu_vol_img:latest   
ni5xrb60u71i   hdp_hdp3      replicated      1/1        mkenjis/ubhdpclu_vol_img:latest
```

6. access hadoop master node
```shell
$ docker container ls   # run it in each node and check which <container ID> is running the hadoop master constainer
CONTAINER ID   IMAGE                             COMMAND                  CREATED              STATUS              PORTS      NAMES
d723786ae3e0   mkenjis/ubhdpclu_vol_img:latest   "/usr/bin/supervisord"   About a minute ago   Up About a minute   9000/tcp   hadoop_hdp3.1.pmvvdxosgi2dkz7m8vi3i0x8t
3895ee795371   mkenjis/ubhdpclu_vol_img:latest   "/usr/bin/supervisord"   About a minute ago   Up About a minute   9000/tcp   hadoop_hdpmst.1.j04grga7ioelyt1vtr26h3fmr

$ docker container exec -it <hdpmst ID> bash
```

7. check HDFS service
```shell
$ hdfs dfsadmin -report
Configured Capacity: 15000010752 (13.97 GB)
Present Capacity: 11198918656 (10.43 GB)
DFS Remaining: 11198906368 (10.43 GB)
DFS Used: 12288 (12 KB)
DFS Used%: 0.00%
Under replicated blocks: 0
Blocks with corrupt replicas: 0
Missing blocks: 0
Missing blocks (with replication factor 1): 0

-------------------------------------------------
Live datanodes (3):

Name: 10.0.3.9:50010 (hadoop_hdp3.1.pmvvdxosgi2dkz7m8vi3i0x8t.hadoop_mynet)
Hostname: d723786ae3e0
Decommission Status : Normal
Configured Capacity: 5000003584 (4.66 GB)
DFS Used: 4096 (4 KB)
Non DFS Used: 1030483968 (982.75 MB)
DFS Remaining: 3969515520 (3.70 GB)
DFS Used%: 0.00%
DFS Remaining%: 79.39%
Configured Cache Capacity: 0 (0 B)
Cache Used: 0 (0 B)
Cache Remaining: 0 (0 B)
Cache Used%: 100.00%
Cache Remaining%: 0.00%
Xceivers: 1
Last contact: Mon Dec 06 17:48:32 CST 2021

Name: 10.0.3.6:50010 (hadoop_hdp2.1.58h0e99ortmxmo4pthah310e0.hadoop_mynet)
Hostname: 384ac655657e
Decommission Status : Normal
Configured Capacity: 5000003584 (4.66 GB)
DFS Used: 4096 (4 KB)
Non DFS Used: 1742635008 (1.62 GB)
DFS Remaining: 3257364480 (3.03 GB)
DFS Used%: 0.00%
DFS Remaining%: 65.15%
Configured Cache Capacity: 0 (0 B)
Cache Used: 0 (0 B)
Cache Remaining: 0 (0 B)
Cache Used%: 100.00%
Cache Remaining%: 0.00%
Xceivers: 1
Last contact: Mon Dec 06 17:48:30 CST 2021

Name: 10.0.3.3:50010 (hadoop_hdp1.1.js26icvnvf6jca9nwjzhg0gwi.hadoop_mynet)
Hostname: 54a27b4ab660
Decommission Status : Normal
Configured Capacity: 5000003584 (4.66 GB)
DFS Used: 4096 (4 KB)
Non DFS Used: 1027973120 (980.35 MB)
DFS Remaining: 3972026368 (3.70 GB)
DFS Used%: 0.00%
DFS Remaining%: 79.44%
Configured Cache Capacity: 0 (0 B)
Cache Used: 0 (0 B)
Cache Remaining: 0 (0 B)
Cache Used%: 100.00%
Cache Remaining%: 0.00%
Xceivers: 1
Last contact: Mon Dec 06 17:48:32 CST 2021
```

8. create HDFS directory and copy files in this directory
```shell
$ hdfs dfs -ls /
$ hdfs dfs -mkdir /data
$ hdfs dfs -ls /
Found 1 items
drwxr-xr-x   - root supergroup          0 2021-12-06 17:49 /data
$ cd $HADOOP_HOME   # it changes to /usr/local/hadoop-2.7.3
$ ls -l
total 108
-rw-r--r-- 1 root root 84854 Aug 17  2016 LICENSE.txt
-rw-r--r-- 1 root root 14978 Aug 17  2016 NOTICE.txt
-rw-r--r-- 1 root root  1366 Aug 17  2016 README.txt
drwxr-xr-x 2 root root   194 Aug 17  2016 bin
drwxr-xr-x 1 root root    20 Aug 17  2016 etc
drwxr-xr-x 2 root root   106 Aug 17  2016 include
drwxr-xr-x 3 root root    20 Aug 17  2016 lib
drwxr-xr-x 2 root root   239 Aug 17  2016 libexec
drwxr-xr-x 2 root root   327 Dec  6 17:47 logs
drwxr-xr-x 2 root root  4096 Aug 17  2016 sbin
drwxr-xr-x 4 root root    31 Aug 17  2016 share
$ hdfs dfs -put *.txt /data
$ hdfs dfs -ls /data
Found 3 items
-rw-r--r--   2 root supergroup      84854 2021-12-06 17:50 /data/LICENSE.txt
-rw-r--r--   2 root supergroup      14978 2021-12-06 17:50 /data/NOTICE.txt
-rw-r--r--   2 root supergroup       1366 2021-12-06 17:50 /data/README.txt
```

9. copy mapper and reducer Python scripts
```shell
$ cd ~  # return to home directory
$ vi mapper.py
$ vi reducer.py
$ find $HADOOP_HOME -name '*streaming*'
/usr/local/hadoop-2.7.3/share/doc/hadoop/api/org/apache/hadoop/streaming
/usr/local/hadoop-2.7.3/share/doc/hadoop/hadoop-streaming
/usr/local/hadoop-2.7.3/share/hadoop/tools/lib/hadoop-streaming-2.7.3.jar
/usr/local/hadoop-2.7.3/share/hadoop/tools/sources/hadoop-streaming-2.7.3-sources.jar
/usr/local/hadoop-2.7.3/share/hadoop/tools/sources/hadoop-streaming-2.7.3-test-sources.jar
```

10. run the wordcount mapreduce example
```shell
$ hadoop jar /usr/local/hadoop-2.7.3/share/hadoop/tools/lib/hadoop-streaming-2.7.3.jar -file mapper.py -mapper mapper.py -file reducer.py -reducer reducer.py -input /data -output /result
21/12/06 17:57:07 WARN streaming.StreamJob: -file option is deprecated, please use generic option -files instead.
21/12/06 17:57:07 WARN util.NativeCodeLoader: Unable to load native-hadoop library for your platform... using builtin-java classes where applicable
packageJobJar: [mapper.py, reducer.py, /tmp/hadoop-unjar4647696537138809119/] [] /tmp/streamjob6959985723008131559.jar tmpDir=null
21/12/06 17:57:08 INFO client.RMProxy: Connecting to ResourceManager at 3895ee795371/10.0.3.12:8040
21/12/06 17:57:09 INFO client.RMProxy: Connecting to ResourceManager at 3895ee795371/10.0.3.12:8040
21/12/06 17:57:10 INFO mapred.FileInputFormat: Total input paths to process : 3
21/12/06 17:57:10 INFO mapreduce.JobSubmitter: number of splits:4
21/12/06 17:57:10 INFO mapreduce.JobSubmitter: Submitting tokens for job: job_1638834441798_0001
21/12/06 17:57:11 INFO impl.YarnClientImpl: Submitted application application_1638834441798_0001
21/12/06 17:57:11 INFO mapreduce.Job: The url to track the job: http://3895ee795371:8088/proxy/application_1638834441798_0001/
21/12/06 17:57:11 INFO mapreduce.Job: Running job: job_1638834441798_0001
21/12/06 17:57:23 INFO mapreduce.Job: Job job_1638834441798_0001 running in uber mode : false
21/12/06 17:57:23 INFO mapreduce.Job:  map 0% reduce 0%
21/12/06 17:57:33 INFO mapreduce.Job:  map 75% reduce 0%
21/12/06 17:57:34 INFO mapreduce.Job:  map 100% reduce 0%
21/12/06 17:57:41 INFO mapreduce.Job:  map 100% reduce 100%
21/12/06 17:57:41 INFO mapreduce.Job: Job job_1638834441798_0001 completed successfully
21/12/06 17:57:41 INFO mapreduce.Job: Counters: 50
        File System Counters
                FILE: Number of bytes read=155599
                FILE: Number of bytes written=920601
                FILE: Number of read operations=0
                FILE: Number of large read operations=0
                FILE: Number of write operations=0
                HDFS: Number of bytes read=105664
                HDFS: Number of bytes written=30052
                HDFS: Number of read operations=15
                HDFS: Number of large read operations=0
                HDFS: Number of write operations=2
        Job Counters 
                Killed map tasks=1
                Launched map tasks=4
                Launched reduce tasks=1
                Data-local map tasks=4
                Total time spent by all maps in occupied slots (ms)=34237
                Total time spent by all reduces in occupied slots (ms)=4812
                Total time spent by all map tasks (ms)=34237
                Total time spent by all reduce tasks (ms)=4812
                Total vcore-milliseconds taken by all map tasks=34237
                Total vcore-milliseconds taken by all reduce tasks=4812
                Total megabyte-milliseconds taken by all map tasks=35058688
                Total megabyte-milliseconds taken by all reduce tasks=4927488
        Map-Reduce Framework
                Map input records=2030
                Map output records=14232
                Map output bytes=127129
                Map output materialized bytes=155617
                Input split bytes=370
                Combine input records=0
                Combine output records=0
                Reduce input groups=2400
                Reduce shuffle bytes=155617
                Reduce input records=14232
                Reduce output records=2400
                Spilled Records=28464
                Shuffled Maps =4
                Failed Shuffles=0
                Merged Map outputs=4
                GC time elapsed (ms)=1237
                CPU time spent (ms)=6780
                Physical memory (bytes) snapshot=1253863424
                Virtual memory (bytes) snapshot=9740931072
                Total committed heap usage (bytes)=945291264
        Shuffle Errors
                BAD_ID=0
                CONNECTION=0
                IO_ERROR=0
                WRONG_LENGTH=0
                WRONG_MAP=0
                WRONG_REDUCE=0
        File Input Format Counters 
                Bytes Read=105294
        File Output Format Counters 
                Bytes Written=30052
21/12/06 17:57:41 INFO streaming.StreamJob: Output directory: /result
```

11. check HDFS result/part-00000 to view the output results.
```shell
$ hdfs dfs -ls /                  
Found 3 items
drwxr-xr-x   - root supergroup          0 2021-12-06 17:50 /data
drwxr-xr-x   - root supergroup          0 2021-12-06 17:57 /result
drwx------   - root supergroup          0 2021-12-06 17:57 /tmp
$ hdfs dfs -ls /result
Found 2 items
-rw-r--r--   2 root supergroup          0 2021-12-06 17:57 /result/_SUCCESS
-rw-r--r--   2 root supergroup      30052 2021-12-06 17:57 /result/part-00000
$ hdfs dfs -cat /result/part-00000 | head -20
""AS    2
"AS     17
"COPYRIGHTS     1
"Contribution"  2
"Contributor"   2
"Derivative     1
"GCC    1
"Legal  1
"License"       1
"License");     2
"Licensed       1
"Licensor"      1
"Losses")       1
"NOTICE"        1
"Not    1
"Object"        1
"Program"       1
"Recipient"     1
"Software"),    5
"Source"        1
```

## Adding a DataNode to Hadoop cluster

in hdpmst :
```shell
copy to new node
> core-site.xml, 
> hdfs-site.xml, 
> mapred-site.xml, 
> yarn-site.xml, 
> hadoop-env.sh
```

in new node:
```shell
run 
$ hadoop-daemon.sh start datanode
$ yarn-daemon.sh start nodemanager
```

in hdpmst :
- check new datanode (hdp4) added to HDP cluster:
```shell

$ hdfs dfsadmin -report
23/01/07 15:10:55 WARN util.NativeCodeLoader: Unable to load native-hadoop library for your platform... using builtin-java classes where applicable
Configured Capacity: 147371851776 (137.25 GB)
Present Capacity: 77290303488 (71.98 GB)
DFS Remaining: 76683739136 (71.42 GB)
DFS Used: 606564352 (578.46 MB)
DFS Used%: 0.78%
Under replicated blocks: 0
Blocks with corrupt replicas: 0
Missing blocks: 0
Missing blocks (with replication factor 1): 0

-------------------------------------------------
Live datanodes (4):

Name: 10.0.1.15:50010 (hdp_hdp4.1.psqz8ee5jxja62fv9a94rqzym.mynet)
Hostname: 08f2305f6d7c
Decommission Status : Normal
Configured Capacity: 5000003584 (4.66 GB)
DFS Used: 4096 (4 KB)
Non DFS Used: 1037230080 (989.18 MB)
DFS Remaining: 3962769408 (3.69 GB)
DFS Used%: 0.00%
DFS Remaining%: 79.26%
Configured Cache Capacity: 0 (0 B)
Cache Used: 0 (0 B)
Cache Remaining: 0 (0 B)
Cache Used%: 100.00%
Cache Remaining%: 0.00%
Xceivers: 1
Last contact: Sat Jan 07 15:10:56 CST 2023


Name: 10.0.1.3:50010 (hdp_hdp2.1.ykf2ptsjtqmw7e89zmfffwu1x.mynet)
Hostname: f5cb08c8a752
Decommission Status : Normal
Configured Capacity: 68685922304 (63.97 GB)
DFS Used: 32735232 (31.22 MB)
Non DFS Used: 34187227136 (31.84 GB)
DFS Remaining: 34465959936 (32.10 GB)
DFS Used%: 0.05%
DFS Remaining%: 50.18%
Configured Cache Capacity: 0 (0 B)
Cache Used: 0 (0 B)
Cache Remaining: 0 (0 B)
Cache Used%: 100.00%
Cache Remaining%: 0.00%
Xceivers: 1
Last contact: Sat Jan 07 15:10:57 CST 2023


Name: 10.0.1.12:50010 (hdp_hdp1.1.xg7nw6p77utzq4he1cnlwr5wm.mynet)
Hostname: 87611be0019f
Decommission Status : Normal
Configured Capacity: 5000003584 (4.66 GB)
DFS Used: 303276032 (289.23 MB)
Non DFS Used: 907677696 (865.63 MB)
DFS Remaining: 3789049856 (3.53 GB)
DFS Used%: 6.07%
DFS Remaining%: 75.78%
Configured Cache Capacity: 0 (0 B)
Cache Used: 0 (0 B)
Cache Remaining: 0 (0 B)
Cache Used%: 100.00%
Cache Remaining%: 0.00%
Xceivers: 1
Last contact: Sat Jan 07 15:10:54 CST 2023


Name: 10.0.1.6:50010 (hdp_hdp3.1.cpyekeee5362n8b16p1y8qakt.mynet)
Hostname: 117231c6a093
Decommission Status : Normal
Configured Capacity: 68685922304 (63.97 GB)
DFS Used: 270548992 (258.02 MB)
Non DFS Used: 33949413376 (31.62 GB)
DFS Remaining: 34465959936 (32.10 GB)
DFS Used%: 0.39%
DFS Remaining%: 50.18%
Configured Cache Capacity: 0 (0 B)
Cache Used: 0 (0 B)
Cache Remaining: 0 (0 B)
Cache Used%: 100.00%
Cache Remaining%: 0.00%
Xceivers: 1
Last contact: Sat Jan 07 15:10:57 CST 2023

```