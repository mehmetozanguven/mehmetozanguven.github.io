---
layout: post
title:  "Apache Flink Series 5 — Create Sample Apache Flink Cluster on Local Machine — Part 1"
date:   2020-02-23 19:45:31 +0530
categories: "apache-flink"
author: "mehmetozanguven"
---

In this post, we are creating simple Flink cluster own local machine.

Before diving into creating cluster, configuration and etc.. let’s summarize **what are the steps to create cluster and deploying job to Flink.**

Typically, when you want to create cluster and submit your job to the Flink, you should follow these steps:

1. Determine the cluster types. (Should it be development mode, standalone mode, or built on cloud ?)
2. Prepare your cluster setup, this setup include configuration setup(make sure that your cluster is up and running)
3. Prepare your stream job (which includes your Java/Scale classes that read from data sources, then apply some transformation and write the result two or more data sinks)
4. Convert your stream job to the .jar file(with mvn clean install)
5. Submit your jar file to the flink cluster (either dashboard or command line)

Don’t forget to download the latest version of Apache Flink https://archive.apache.org/dist/flink/flink-1.10.0/ . If you are going to download source file, you will need to do additional things. My advice download the apache-flink-1.10.0.tar.gz file and use it directly.

## Step 1 Determine the cluster types

We have 3 options:
1. Local
2. Cluster (standalone, YARN)
3. Cloud (GCE, EC2)

For our example, I will use option 2 with standalone(I don’t know YARN & Hadoop environment very well). Option 1 is not for real cases, for option 3, I don’t know how GCE and EC2 works.

You may ask how you are going to form a cluster with a single machine? For our example, JobManager and TaskManager(we will have one) will run the on same machine.

> Note: Ideally you should have one flink process instance per machines. Other solutions (like I said before) may lead(most probably) to memory problem(insufficient memory exception).

Now, let me talk about the standalone mode a little bit:
- Standalone setup expects that cluster consists of one master node and one or more worker nodes. And each node must have Java 1.8++ and ssh (sshd must be running on each node, therefore all nodes can communicate each other. In our example, we don’t need it.)
- JAVA_HOME environment variable must be set on each node. Flink will use it.
- Installation path for Apache Flink must be the same for each node/machine.

After downloading Flink, go to the your flink path and go to deps/bin folder:

```bash
$ cd pathToFlink/apache-flink-1.10.0/bin
```

Then start the script called start-cluster.sh

```bash
$ ./start-cluster.sh
Starting cluster.
Starting standalonesession daemon on host Mehmets-MacBook-Pro.local.
Starting taskexecutor daemon on host Mehmets-MacBook-Pro.local.
```

After that open the address localhost:8081, and you should see the screen like this one:

<img src="/assets/apache_flink/flink_dashboard.png" alt="flink_dashboard.png" title="Flink Dashboard" />

**Now you are ready to go, your flink cluster is up and running.**

To stop Flink (and cluster also), run the script in the bin folder:

```bash
$ ./stop-cluster.sh
Stopping taskexecutor daemon (pid: 94909) on host Mehmets-MacBook-Pro.local.
Stopping standalonesession daemon (pid: 94648) on host Mehmets-MacBook-Pro.local.
```

And also make sure that all processes about Flink is not running. You can check the running processes about flink via:

```bash
$ ps -ef | grep flink
```

<br />

## Step 2 Prepare Cluster Setup

You may ask how flink knows taskmanager, jobmanager address and also how it knows that we are going to create jobmanager and taskmanager on the same machine?

This setup part will answer these questions and more. But before that let’s point to the some important files in the Flink folder.

**bin** folder contains the scripts for creating cluster or stopping, starting taskmanager, jobmanager and etc...

**log** folder contains log file for taskmanager if machine is a taskmanager or jobmanager if machine is a jobmanager. For our example, it contains logs for both taskmanager and jobmanager.

**lib** folder contains additional jar files that flink will need at runtime.

**conf** folder contains configuration files for flink, log.properties, zookeeper setup etc…

Most of the time you are going to deal with conf folder and your stream job.

**conf** folder includes the following files:
- **flink-conf.yaml** => holds the flink configuration. For example, you can set taskamanger memory, state backend type (rocksdb, memory etc..), parallelism of the job and more.
- **log4j-cli/console/yarn.properties** => Flink uses log4j for logging mechanism as default. These files are related to logging mechanism
- **logback-console/yarn.xml** => Flink also support logback if you want to use. (if you don’t delete log4j files, logback files will have no effect) These files are related to logging mechanism for logback.
- **masters** => includes ip addresses of masters nodes.
- **slaves** => includes ip addresses of slaves nodes.
- **sql-client-defaults.yaml** => includes configuration for Flink’s SQL client
- **zoo.cfg** => holds the configuration for Apache ZooKeeper, if you want to use.

Let’s look at the default configuration:
- **flink-conf.yaml (note that I removed the comment lines):** When we read this file, we see that jobmanager will start in the address localhost:6123

```log
# JobManager ip address to communicate with it. Use this key if you have one master node with static location. Don't use it for highly available system.

jobmanager.rpc.address: localhost

# Determine the communication port for JobManager

jobmanager.rpc.port: 6123

# The heap size for the JobManager JVM 

jobmanager.heap.size: 1024m

# The total process memory size for the TaskManager.
#
# Note this accounts for all memory usage within the TaskManager process, including JVM metaspace and other overhead.

taskmanager.memory.process.size: 1568m

# The number of task slots that each TaskManager offers. Each slot runs one parallel pipeline. This number is related to the CPU number of you machine. If your machine has 16 CPUs, you can write 16 for this key.

taskmanager.numberOfTaskSlots: 1

# The parallelism used for programs that did not specify and other parallelism. When you deploy your stream job, stream job will have parallelism of 1 by default, you can re-write this key when deploying job from dashboard.
# However be careful that stream parallelism is not higher than the total number taskmanager slot

parallelism.default: 1
```

- **slaves**: When we read this file, we see that there will be taskmanager in the ip address localhost:

```log
# this file contains the address of taskmanager
# in this case, when we say ./start-cluster.sh, 
# this will trigger to start taskmanager in the ip address localhost

localhost
```

Overall, we will have **JobManager (localhost:6123) and TaskManager(localhost) on the machine.**

Let’s modify the configuration and see the changes (make sure that you stopped to cluster)

Learn your cpu number and update this line in the flink-conf.yaml (my computer has 16 CPUs)

```log
...
# now i can run 16 parallel works at the same
taskmanager.numberOfTaskSlots: 16
...
```

Then, start your cluster and open the dashboard:

<img src="/assets/apache_flink/task_manager_with_16.png" alt="task_manager_with_16.png" title="TaskManager with 16 parallel tasks" />

I will continue with part 2 from step 3

Last but not least, wait for the next post …