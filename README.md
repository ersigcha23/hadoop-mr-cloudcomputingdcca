# Running the Word count example on a Hadoop Cluster in Docker

This directory encompasses all the essential files for deploying an Hadoop cluster using Docker. The Hadoop cluster runs the Word Count example using Bash scripts for the Mapper and Reducer functions.

It is assumed that the host computer has Docker installed.

The next steps must be carried out:

### 1. Clone the base repository

```bash
git clone https://github.com/MartinCastroAlvarez/hadoop-hdfs-map-reduce-docker
```

### 2. Navigating to the cloned repo

```bash
cd hadoop-hdfs-map-reduce-docker
```

### 3. Creating a virtual network in Docker

This step is required since the Dockerfile from the project doesn't create a network for interconecting the containers. The virtual network must be named "hbase"

```bash
docker network create hbase
```

### 4. Starting the Hadoop ecosystem

First, in order to avoid getting errors, it is recommended to check and delete existing containers from past executions of this tutorial. 

```bash
docker rm -f $(docker ps -a -q)
docker volume rm $(docker volume ls -q)
```

After the verification, the Hadoop cluster is generated:

```bash
docker-compose up
```

The status of the Docker cluster cna be verified with:

```bash
docker ps
```

### 5. Accessing the namenode and creating a HDFS directory

The main node of the cluster is the namenode, which can be accessed with the following command:

```bash
docker exec -it namenode /bin/bash
```

In order to execute the Word Count example, a HDFS directory must be created with the following command within the namenode:

```bash
hdfs dfs -mkdir -p /user/root
```

The list of existing directories within the HDFS can be visualized with the following command:

```bash
hdfs dfs -ls /
```

You can exit fromt he namenode with the command:

```bash
exit
```


### 6. Generating text files for the WordCount execution with Python

The repository includes a Python code to generate textfiles with randomly generated words. It is recommeded to generate a Python Virtual Environment to run this Python application:

#### - Activate the virtual environment

##### Windows
```bash
.env/Scripts/activate
```

##### Linux
```bash
source .env/bin/activate
```

#### - Install the required libraries within the Virtual Environment

```bash
pip install requests
```

```bash
pip install hdfs
```

#### - Run the text files generator
This Python programm generates and sends various text files to the Hadoop cluster for the word count app. The app uses the HDFS library to establish a connection with the Hadoop cluster.

```bash
python3 app/test_hdfs.py
```

### 7 - Uploading the mapper and reducer jobs/code to the Docker container
This tutorial uses bash scripts as mapper and reducer functions. Both scripts are included in the repository files within the app folder. However, the code within the reducer file must be replaced to obtained the correct results. The code of the reducer is as follows:

```bash
#!/bin/bash

currkey=""
currcount=0
while IFS=$'\t' read -r key val
do
  if [[ $key == $currkey ]]
  then
      currcount=$(( currcount + val ))
  else
    if [ -n "$currkey" ]
    then
      echo -e ${currkey} "\t" ${currcount} 
    fi
    currkey=$key
    currcount=1
  fi
done
# last one
echo -e ${currkey} "\t" ${currcount}
```

Next, the mapper and reducer can be transferred to the namenode with the following commands:

```console
docker cp ./app/mapper.sh namenode:/tmp/mapper.sh
docker cp ./app/reducer.sh namenode:/tmp/reducer.sh
```

In this case, the mapper and reducer files are copied to the /tmp directory within the namenode.

### 8 - Testing the mapper and the reducer within the namenode container

First, the namenode container is accessed with the following command:

```console
docker exec -it namenode /bin/bash
```

Next, the mapper job can be tested with:

```console
echo "asdf asdf asdkfjh 99asdf asdf" | /tmp/mapper.sh
```

The following output will appear in the terminal:

```console
asdf	1
asdf	1
asdkfjh	1
99asdf	1
asdf	1
```

The reducer job can be tested with:

```console
echo "asdf 1 1 2 3" | /tmp/reducer.sh
```

This generates the following output:

```console
asdf	7
```

#### - Error bad interpreter: No such file or directory
If you are using Windows to run the tutorial, when the files are sent to the namenode container, some conversion characters are added in the breaklines. This causes the Bad Interpreter error.
In this case, it is required to remove this special characters using the following commands:

```console
sed 's/\r$//' /tmp/mapper.sh > /tmp/mapper-rev.sh
rm /tmp/mapper.sh
mv /tmp/mapper-rev.sh /tmp/mapper.sh
chmod 755 /tmp/mapper.sh
```

```console
sed 's/\r$//' /tmp/reducer.sh > /tmp/reducer-rev.sh
rm /tmp/reducer.sh
mv /tmp/reducer-rev.sh /tmp/reducer.sh
chmod 755 /tmp/reducer.sh 
```

### 9 - Creating the directory for the MapReduce job

```console
hadoop fs -mkdir /jobs
```

### 10 - Uploading the map and reduce job to the HDFS


#### - Remove files from previous executions
```console
hadoop fs -rm /jobs/mapper.sh
hadoop fs -rm /jobs/reducer.sh
```

#### - Put the map and reduce jobs in the HDFS
```console
hadoop fs -put /tmp/mapper.sh /jobs/mapper.sh
hadoop fs -put /tmp/reducer.sh /jobs/reducer.sh
```

#### - Set execution privileges within the HDFS
```console
hadoop fs -chmod 555 /jobs/mapper.sh
hadoop fs -chmod 555 /jobs/reducer.sh
```

#### - List jobs
```console
hadoop fs -ls /jobs/
```

   The following output is expected:
```console
\Found 2 items
-r-xr-xr-x   3 root supergroup         91 2024-02-26 05:27 /jobs/mapper.sh
-r-xr-xr-x   3 root supergroup        141 2024-02-26 05:27 /jobs/reducer.sh
```

### 11 - Running the MapReduce job

#### - Checking that the output files directory is empty
```console
hadoop fs -rm -r /jobs/output
```

#### - Running MapReduce
```console
hadoop jar \
    /opt/hadoop-3.2.1/share/hadoop/tools/lib/hadoop-streaming-3.2.1.jar \
    -files "/tmp/mapper.sh,/tmp/reducer.sh" \
    -input /user/martin/*.txt \
    -output /jobs/output \
    -mapper "mapper.sh" \
    -reducer "reducer.sh"
```

An output like this must be generated:
```console
2024-02-26 05:35:57,638 INFO mapreduce.Job: Counters: 54
        File System Counters
                FILE: Number of bytes read=311963
                FILE: Number of bytes written=62392647
                FILE: Number of read operations=0
                FILE: Number of large read operations=0
                FILE: Number of write operations=0
                HDFS: Number of bytes read=444766
                HDFS: Number of bytes written=548501
                HDFS: Number of read operations=797
                HDFS: Number of large read operations=0
                HDFS: Number of write operations=2
                HDFS: Number of bytes read erasure-coded=0
        Job Counters
                Launched map tasks=264
                Launched reduce tasks=1
                Rack-local map tasks=264
                Total time spent by all maps in occupied slots (ms)=1953584
                Total time spent by all reduces in occupied slots (ms)=44584
                Total time spent by all map tasks (ms)=488396
                Total time spent by all reduce tasks (ms)=5573
                Total vcore-milliseconds taken by all map tasks=488396
                Total vcore-milliseconds taken by all reduce tasks=5573
                Total megabyte-milliseconds taken by all map tasks=2000470016
                Total megabyte-milliseconds taken by all reduce tasks=45654016
        Map-Reduce Framework
                Map input records=264
                Map output records=64515
                Map output bytes=548501
                Map output materialized bytes=389116
                Input split bytes=25559
                Combine input records=0
                Combine output records=0
                Reduce input groups=54594
                Reduce shuffle bytes=389116
                Reduce input records=64515
                Reduce output records=64515
                Spilled Records=129030
                Shuffled Maps =264
                Failed Shuffles=0
                Merged Map outputs=264
                GC time elapsed (ms)=9008
                CPU time spent (ms)=118820
                Physical memory (bytes) snapshot=77591416832
                Virtual memory (bytes) snapshot=1359154561024
                Total committed heap usage (bytes)=73203712000
                Peak Map Physical memory (bytes)=328732672
                Peak Map Virtual memory (bytes)=5121912832
                Peak Reduce Physical memory (bytes)=203116544
                Peak Reduce Virtual memory (bytes)=8461942784
        Shuffle Errors
                BAD_ID=0
                CONNECTION=0
                IO_ERROR=0
                WRONG_LENGTH=0
                WRONG_MAP=0
                WRONG_REDUCE=0
        File Input Format Counters
                Bytes Read=419207
        File Output Format Counters
                Bytes Written=548501
2024-02-26 05:35:57,638 INFO streaming.StreamJob: Output directory: /jobs/output
```

#### - Checking a generated test file within HDFS
```console
hdfs dfs -tail /user/martin/doc-774.txt
```

### 12 - Visualizing the History server web interface

Access through a web browser to the following URL: [http://127.0.0.1:8188/applicationhistory](http://127.0.0.1:8188/applicationhistory)


### 13 - Checking results of the MapReduce job

#### - Listing generated files
```console
hadoop fs -ls /jobs/output/
```

#### - Printing the output of the MapReduce job
```console
hadoop fs -cat /jobs/output/part-00000
```

#### - Extracting the results from the Hadoop cluster
```console
hdfs dfs -cat /jobs/output/part-00000 > /tmp/result_wc.txt
```

#### - Exiting from the namenode
```console
exit
```

#### - Copying the result file from Docker to the current directory
```console
docker cp namenode:/tmp/result_wc.txt .
```

### 14 - Turning down the containers
```console
docker-compose down
```

#### - Removing the containers and volumes
```console
docker rm -f $(docker ps -a -q)
docker volume rm $(docker volume ls -q)
```
