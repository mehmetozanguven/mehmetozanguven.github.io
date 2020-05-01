---
layout: post
title:  "Apache Flink Series 6 —Reading the Log files"
date:   2020-03-05 23:30:31 +0530
categories: "apache-flink"
author: "mehmetozanguven"
---

In this post, we will look at the log files (both for TaskManager and JobManager) and try to understand what is going on Flink cluster.

Actually this post will be about the step 3 for creating sample Flink cluster. However, I just thought that reading logs files would be a great improvement to understand basic of any tools/framework etc.. Therefore I decided to write about post for that.

From the previous blog, we could run simple flink cluster with one JobManager along with a TaskManager.

Let’s re-process this, with only one task slot (setting the numberOfTaskSlots to 1 in the flink-conf.yaml file):

```log
taskmanager.numberOfTaskSlots: 1
```

Now run your cluster with: `$ pathToFlink/bin/start-cluster.sh`

> Note: If you run flink cluster before, you can delete logs from the previous setup. Otherwise flink will append the newly created log to the existing ones

## JobManager’s Log File

```log
2020-02-28 00:03:44,009 INFO  org.apache.flink.runtime.entrypoint.ClusterEntrypoint         -  Starting StandaloneSessionClusterEntrypoint (Version: 1.10.0, Rev:aa4eb8f, Date:07.02.2020 @ 19:18:19 CET)
```

Because we are creating Standalone cluster, Flink will also create the appropriate classes for Standalone. If you look at the Flink source, you will see that **ClusterEntrypoint is a abstract class and there is a class called StandaloneSessionClusterEntrypoint which extends it.**

```log
(1)
2020-02-28 00:03:44,009 INFO  org.apache.flink.runtime.entrypoint.ClusterEntrypoint         -  OS current user: mehmetozanguven

(2)
2020-02-28 00:03:44,010 INFO  org.apache.flink.runtime.entrypoint.ClusterEntrypoint         -  Current Hadoop/Kerberos user: <no hadoop dependency found>

(3)
2020-02-28 00:03:44,010 INFO  org.apache.flink.runtime.entrypoint.ClusterEntrypoint         -  JVM: OpenJDK 64-Bit Server VM - AdoptOpenJDK - 11/11.0.6+10

(4)
2020-02-28 00:03:44,010 INFO  org.apache.flink.runtime.entrypoint.ClusterEntrypoint         -  Maximum heap size: 1024 MiBytes

(5)
2020-02-28 00:03:44,010 INFO  org.apache.flink.runtime.entrypoint.ClusterEntrypoint         -  JAVA_HOME: /home/mehmetozanguven/JavaJDKS/jdk-11.0.6+10
```

Logs that gives information(the environment, like code revision, current user, Java version and JVM parameters), like the above will be print in the method:

```java
org.apache.flink.runtime.util.EnvironmentInformation.logEnvironmentInfo(...)
```

For example java_home component is reading like this

```java
logEnvironmentInfo(...) {
    // ...
    String javaHome = System.getenv("JAVA_HOME");
    //...
}
```

And all the information about system, environment etc... will be print in here:

```java
logEnvironmentInfo(...){
    log.info("--------------------------------------------------------------------------------");
    log.info(" Starting " + componentName + " (Version: " + version + ", "
      + "Rev:" + rev.commitId + ", " + "Date:" + rev.commitDate + ")");
    log.info(" OS current user: " + System.getProperty("user.name"));
    log.info(" Current Hadoop/Kerberos user: " + getHadoopUser());
    log.info(" JVM: " + jvmVersion);
    log.info(" Maximum heap size: " + maxHeapMegabytes + " MiBytes");
    log.info(" JAVA_HOME: " + (javaHome == null ? "(not set)" : javaHome));
}
```

After that we see these logs in the JobManager’s log file:

```log
2020-02-28 00:03:44,010 INFO  org.apache.flink.runtime.entrypoint.ClusterEntrypoint         -  JVM Options:

2020-02-28 00:03:44,010 INFO  org.apache.flink.runtime.entrypoint.ClusterEntrypoint         -     -Xms1024m

2020-02-28 00:03:44,010 INFO  org.apache.flink.runtime.entrypoint.ClusterEntrypoint         -     -Xmx1024m

2020-02-28 00:03:44,011 INFO  org.apache.flink.runtime.entrypoint.ClusterEntrypoint         -     -Dlog.file=/home/mehmetozanguven/Desktop/ApacheTools/flink-1.10.0/log/flink-mehmetozanguven-standalonesession-0-mehmetozanguven-ABRA-A5-V5.log

2020-02-28 00:03:44,011 INFO  org.apache.flink.runtime.entrypoint.ClusterEntrypoint         -     -Dlog4j.configuration=file:/home/mehmetozanguven/Desktop/ApacheTools/flink-1.10.0/conf/log4j.properties

2020-02-28 00:03:44,011 INFO  org.apache.flink.runtime.entrypoint.ClusterEntrypoint         -     -Dlogback.configurationFile=file:/home/mehmetozanguven/Desktop/ApacheTools/flink-1.10.0/conf/logback.xml
```

Nothing but Flink reads JVM options that passed on the startup, you may think that “we just started the cluster via .sh file who passed the -Dlog.file/log4j etc.. ?”

These system properties is passed by another .sh file called `config.sh`

This file looks at the flink-conf.yaml, if it finds the key value parameter for any of these system properties, it will use it, otherwise default value will be used.

```log
2020-02-28 00:03:44,011 INFO  org.apache.flink.runtime.entrypoint.ClusterEntrypoint         -  Program Arguments:

2020-02-28 00:03:44,011 INFO  org.apache.flink.runtime.entrypoint.ClusterEntrypoint         -     --configDir

2020-02-28 00:03:44,011 INFO  org.apache.flink.runtime.entrypoint.ClusterEntrypoint         -     /home/mehmetozanguven/Desktop/ApacheTools/flink-1.10.0/conf

2020-02-28 00:03:44,011 INFO  org.apache.flink.runtime.entrypoint.ClusterEntrypoint         -     --executionMode

2020-02-28 00:03:44,011 INFO  org.apache.flink.runtime.entrypoint.ClusterEntrypoint         -     cluster

2020-02-28 00:03:44,011 INFO  org.apache.flink.runtime.entrypoint.ClusterEntrypoint         -  Classpath: /home/mehmetozanguven/Desktop/ApacheTools/flink-1.10.0/lib/flink-table_2.11-1.10.0.jar:/home/mehmetozanguven/Desktop/ApacheTools/flink-1.10.0/lib/flink-table-blink_2.11-1.10.0.jar:/home/mehmetozanguven/Desktop/ApacheTools/flink-1.10.0/lib/log4j-1.2.17.jar:/home/mehmetozanguven/Desktop/ApacheTools/flink-1.10.0/lib/slf4j-log4j12-1.7.15.jar:/home/mehmetozanguven/Desktop/ApacheTools/flink-1.10.0/lib/flink-dist_2.11-1.10.0.jar:::
```

Flink reads the program arguments (command line arguments) and prints it if found. Then Flink will also print the classpath to execute java classes. These information is also printted in the EnvironmentInformation.logEnvironmentInfo(…) method

```java
if (commandLineArgs == null || commandLineArgs.length == 0) {
   log.info(" Program Arguments: (none)");
}
else {
   log.info(" Program Arguments:");
   for (String s: commandLineArgs) {
      log.info("    " + s);
   }
}

log.info(" Classpath: " + System.getProperty("java.class.path"));

log.info("--------------------------------------------------------------------------------");
```

As also mentioned before these parameter was set by config.sh file

```log
2020-02-28 00:03:44,029 INFO  org.apache.flink.configuration.GlobalConfiguration            - Loading configuration property: jobmanager.rpc.address, localhost

2020-02-28 00:03:44,029 INFO  org.apache.flink.configuration.GlobalConfiguration            - Loading configuration property: jobmanager.rpc.port, 6123

2020-02-28 00:03:44,029 INFO  org.apache.flink.configuration.GlobalConfiguration            - Loading configuration property: jobmanager.heap.size, 1024m

2020-02-28 00:03:44,029 INFO  org.apache.flink.configuration.GlobalConfiguration            - Loading configuration property: taskmanager.memory.process.size, 1568m

2020-02-28 00:03:44,029 INFO  org.apache.flink.configuration.GlobalConfiguration            - Loading configuration property: taskmanager.numberOfTaskSlots, 1

2020-02-28 00:03:44,029 INFO  org.apache.flink.configuration.GlobalConfiguration            - Loading configuration property: parallelism.default, 1

2020-02-28 00:03:44,030 INFO  org.apache.flink.configuration.GlobalConfiguration            - Loading configuration property: jobmanager.execution.failover-strategy, region
```

Flink reads the keys from conf/flink-conf.yaml and print the values.

Here is the one important log line that placed almost at the bottom:

```log
2020-02-28 00:03:46,412 INFO  org.apache.flink.runtime.resourcemanager.StandaloneResourceManager  - Registering TaskManager with ResourceID c1438b06b128b7faf16b60e06f404f05 (akka.tcp://flink@127.0.1.1:36981/user/taskmanager_0) at ResourceManager
```

This line basically says that “TaskManager whose id **c1438b06b128b7faf16b60e06f404f05** registers itself to the ResourceManager.”

After that JobManager requests a task slot from ResourceManager to execute client’s tasks(streams actually).

<br />

## TaskManager’s Log File

```log
2020-02-28 00:03:44,843 INFO  org.apache.flink.runtime.taskexecutor.TaskManagerRunner       -  Starting TaskManager (Version: 1.10.0, Rev:aa4eb8f, Date:07.02.2020 @ 19:18:19 CET)
```

TaskManagerRunner is the entry point for the TaskManager in standalone mode. This class constructs the related components such as memory manager, network, I/O manager etc…

```log
2020-02-28 00:03:45,846 INFO  org.apache.flink.runtime.taskexecutor.TaskManagerRunner       - Starting TaskManager with ResourceID: c1438b06b128b7faf16b60e06f404f05
```

TaskManager id (called ResourceId) is generated in the method:

```java
public final class 
org.apache.flink.runtime.clusterframework.types.ResourceId ... {/**
 * Generate a random resource id.
 *
 * @return A random resource id.
 */
    public static ResourceID generate() {
        return new ResourceID(new AbstractID().toString());
    }
}
```

Let’s look at the logs statement where TaskManager registers itself to the resource manager:

```log
2020-02-28 00:03:46,176 INFO  org.apache.flink.runtime.taskexecutor.TaskExecutor            - Connecting to ResourceManager akka.tcp://flink@localhost:6123/user/resourcemanager(00000000000000000000000000000000).

2020-02-28 00:03:46,336 INFO  org.apache.flink.runtime.taskexecutor.TaskExecutor            - Resolved ResourceManager address, beginning registration

2020-02-28 00:03:46,336 INFO  org.apache.flink.runtime.taskexecutor.TaskExecutor            - Registration at ResourceManager attempt 1 (timeout=100ms)

2020-02-28 00:03:46,426 INFO  org.apache.flink.runtime.taskexecutor.TaskExecutor            - Successful registration at resource manager akka.tcp://flink@localhost:6123/user/resourcemanager under registration id b266a3eeb6c19ada5969142c1ffb2651.
```

Logs clearly indicates that “what is going on”, I suppose.

That’s for this post. Hopefully I will go with part 2 for Creating Sample Apache Flink Cluster.

Last but not least, wait for the next post …