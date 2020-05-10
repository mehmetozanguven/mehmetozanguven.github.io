---
layout: post
title:  "Apache Flink Series 10 -  Reading Log files for State Example"
date:   2020-05-10 14:30:31 +0530
categories: "apache-flink"
author: "mehmetozanguven"
---

In this post, I am going to read the log files from the application that I created in previous post. Here is the github [link](https://github.com/mehmetozanguven/flink_examples/tree/master/StateExample) and also previous post [link](https://mehmetozanguven.github.io/apache-flink/2020/05/02/state-backend-and-state-example.html)

## One Little Change

Before reading, I updated the `taskmanager.numberOfTaskSlots` config. Because one slot offered by TaskManager was not the matching with the real scenarios.

```yaml
# ...
taskmanager.numberOfTaskSlots: 8
# ...
```



To remember how standalone cluster works you may refer to **Apache Flink Series 9 - How Flink & Standalone Cluster Setup Work?**

<br />

<br />

## Plan Visualization for Our Job

- This visualization is done by Flink.

<img src="/assets/apache_flink/plan_visualization.png" alt="plan_visualization.png" title="Plan Visualization for State Example" />

<br />

<br />

Let's start to read log files.

## JobManager's Log File

- Everything starts with `ClusterEntryPoint` which is the base class for the Flink cluster
- Because we used Standalone Cluster setup, Flink used specific concrete implementation of `ClusterEntryPoint` class which is `StandaloneSessionClusterEntrypoint`

```reStructuredText
2020-05-03 22:48:42,350 INFO  org.apache.flink.runtime.entrypoint.ClusterEntrypoint         -  Starting StandaloneSessionClusterEntrypoint (Version: 1.10.0, Rev:aa4eb8f, Date:07.02.2020 @ 19:18:19 CET)
```

<br />

<br />

### How StandaloneSessionClusterEntrypoint runs ?

- Main method of `StandaloneSessionClusterEntrypoint`:

```java
public class StandaloneSessionClusterEntrypoint extends SessionClusterEntrypoint {

 public StandaloneSessionClusterEntrypoint(Configuration configuration) {
  super(configuration);
 }
 
 public static void main(String[] args) {
  // print information about the environment 
  EnvironmentInformation.logEnvironmentInfo(LOG, StandaloneSessionClusterEntrypoint.class.getSimpleName(), args);
     // This signal handler / signal logger is based on Apache Hadoop's org.apache.hadoop.util.SignalLogger.
  SignalHandler.register(LOG);
     // install the safe guard shutdown hook. JVM shotdown hooks are a special construct that allows developers to put some code to be executed when the JVM is shutting down.
  JvmShutdownSafeguard.installAsShutdownHook(LOG);
     // EntrypointClusterConfiguration is class that parsers the command line argument

  EntrypointClusterConfiguration entrypointClusterConfiguration = null;
  final CommandLineParser < EntrypointClusterConfiguration > commandLineParser = new CommandLineParser < > (new EntrypointClusterConfigurationParserFactory());

  try {
   entrypointClusterConfiguration = commandLineParser.parse(args);
  } catch (FlinkParseException e) {
   LOG.error("Could not parse command line arguments {}.", args, e);
   commandLineParser.printHelp(StandaloneSessionClusterEntrypoint.class.getSimpleName());
   System.exit(1);
  }
     // loading configuration from flink-conf.yaml file

  Configuration configuration = loadConfiguration(entrypointClusterConfiguration);
     // create the StandaloneSessionClusterEntrypoint instance 

  StandaloneSessionClusterEntrypoint entrypoint = new StandaloneSessionClusterEntrypoint(configuration);
     // run the cluster with new cluster entry point

  ClusterEntrypoint.runClusterEntrypoint(entrypoint);
 }
}
```

- Let's back to the Job Manager's log file..
- The Logs statements below show the environment that flink runs on it.

````reStructuredText
2020-05-03 22:48:42,351 INFO  org.apache.flink.runtime.entrypoint.ClusterEntrypoint         -  OS current user: mehmetozanguven
2020-05-03 22:48:42,351 INFO  org.apache.flink.runtime.entrypoint.ClusterEntrypoint         -  Current Hadoop/Kerberos user: <no hadoop dependency found>
2020-05-03 22:48:42,351 INFO  org.apache.flink.runtime.entrypoint.ClusterEntrypoint         -  JVM: OpenJDK 64-Bit Server VM - AdoptOpenJDK - 11/11.0.6+10
2020-05-03 22:48:42,351 INFO  org.apache.flink.runtime.entrypoint.ClusterEntrypoint         -  Maximum heap size: 1024 MiBytes
2020-05-03 22:48:42,351 INFO  org.apache.flink.runtime.entrypoint.ClusterEntrypoint         -  JAVA_HOME: /home/mehmetozanguven/JavaJDKS/jdk-11.0.6+10
2020-05-03 22:48:42,352 INFO  org.apache.flink.runtime.entrypoint.ClusterEntrypoint         -  No Hadoop Dependency available
2020-05-03 22:48:42,352 INFO  org.apache.flink.runtime.entrypoint.ClusterEntrypoint         -  JVM Options:
2020-05-03 22:48:42,352 INFO  org.apache.flink.runtime.entrypoint.ClusterEntrypoint         -     -Xms1024m
2020-05-03 22:48:42,352 INFO  org.apache.flink.runtime.entrypoint.ClusterEntrypoint         -     -Xmx1024m
2020-05-03 22:48:42,352 INFO  org.apache.flink.runtime.entrypoint.ClusterEntrypoint         -     -Dlog.file=/home/mehmetozanguven/Desktop/ApacheTools/flink-1.10.0/log/flink-mehmetozanguven-standalonesession-0-mehmetozanguven-ABRA-A5-V5.log
2020-05-03 22:48:42,352 INFO  org.apache.flink.runtime.entrypoint.ClusterEntrypoint         -     -Dlog4j.configuration=file:/home/mehmetozanguven/Desktop/ApacheTools/flink-1.10.0/conf/log4j.properties
2020-05-03 22:48:42,352 INFO  org.apache.flink.runtime.entrypoint.ClusterEntrypoint         -     -Dlogback.configurationFile=file:/home/mehmetozanguven/Desktop/ApacheTools/flink-1.10.0/conf/logback.xml
2020-05-03 22:48:42,352 INFO  org.apache.flink.runtime.entrypoint.ClusterEntrypoint         -  Program Arguments:
2020-05-03 22:48:42,352 INFO  org.apache.flink.runtime.entrypoint.ClusterEntrypoint         -     --configDir
2020-05-03 22:48:42,352 INFO  org.apache.flink.runtime.entrypoint.ClusterEntrypoint         -     /home/mehmetozanguven/Desktop/ApacheTools/flink-1.10.0/conf
2020-05-03 22:48:42,352 INFO  org.apache.flink.runtime.entrypoint.ClusterEntrypoint         -     --executionMode
2020-05-03 22:48:42,352 INFO  org.apache.flink.runtime.entrypoint.ClusterEntrypoint         -     cluster
2020-05-03 22:48:42,352 INFO  org.apache.flink.runtime.entrypoint.ClusterEntrypoint         -  Classpath: /home/mehmetozanguven/Desktop/ApacheTools/flink-1.10.0/lib/flink-table_2.11-1.10.0.jar:/home/mehmetozanguven/Desktop/ApacheTools/flink-1.10.0/lib/flink-table-blink_2.11-1.10.0.jar:/home/mehmetozanguven/Desktop/ApacheTools/flink-1.10.0/lib/log4j-1.2.17.jar:/home/mehmetozanguven/Desktop/ApacheTools/flink-1.10.0/lib/slf4j-log4j12-1.7.15.jar:/home/mehmetozanguven/Desktop/ApacheTools/flink-1.10.0/lib/flink-dist_2.11-1.10.0.jar:::
2020-05-03 22:48:42,353 INFO  org.apache.flink.runtime.entrypoint.ClusterEntrypoint         - 
````



- Log for signal handler:

```reStructuredText
2020-05-03 22:48:42,354 INFO  org.apache.flink.runtime.entrypoint.ClusterEntrypoint         - Registered UNIX signal handlers for [TERM, HUP, INT]
```



- Log from the `flink-conf.yaml file`, `GlobalConfiguration` class loads that file.

```reStructuredText
2020-05-03 22:48:42,377 INFO  org.apache.flink.configuration.GlobalConfiguration            - Loading configuration property: jobmanager.rpc.address, localhost
2020-05-03 22:48:42,378 INFO  org.apache.flink.configuration.GlobalConfiguration            - Loading configuration property: jobmanager.rpc.port, 6123
2020-05-03 22:48:42,378 INFO  org.apache.flink.configuration.GlobalConfiguration            - Loading configuration property: jobmanager.heap.size, 1024m
2020-05-03 22:48:42,378 INFO  org.apache.flink.configuration.GlobalConfiguration            - Loading configuration property: taskmanager.memory.process.size, 1568m
2020-05-03 22:48:42,378 INFO  org.apache.flink.configuration.GlobalConfiguration            - Loading configuration property: taskmanager.numberOfTaskSlots, 8
...
```

<br />

<br />

### How GlobalConfiguration loads the flink-conf.yaml file?

```java
public class StandaloneSessionClusterEntrypoint extends SessionClusterEntrypoint{
    // ...
    public static void main(String[] args) {
    	// ...
        Configuration configuration = loadConfiguration(entrypointClusterConfiguration);
    }
}
// ...
public abstract class ClusterEntrypoint implements AutoCloseableAsync, FatalErrorHandler {
	protected static Configuration loadConfiguration(EntrypointClusterConfiguration entrypointClusterConfiguration) {
    	// ...
        final Configuration configuration = GlobalConfiguration.loadConfiguration(entrypointClusterConfiguration.getConfigDir(), dynamicProperties);
        // ...
    }
}
// ...
public final class GlobalConfiguration {
	public static Configuration loadConfiguration(final String configDir, @Nullable final Configuration dynamicProperties) {
    	// ...
        final File yamlConfigFile = new File(confDirFile, FLINK_CONF_FILENAME);
        // ...
        // code statement where flink-conf.yaml file loaded
        Configuration configuration = loadYAMLResource(yamlConfigFile);
    }
}
```



- At that point, our cluster is starting:

````reStructuredText
2020-05-03 22:48:42,409 INFO  org.apache.flink.runtime.entrypoint.ClusterEntrypoint         - Starting StandaloneSessionClusterEntrypoint.
````

<br />

<br />

### How cluster is starting?

```java
public class StandaloneSessionClusterEntrypoint extends SessionClusterEntrypoint{
    // ...
    public static void main(String[] args) {
    	// ...
        ClusterEntrypoint.runClusterEntrypoint(entrypoint);
    }
}
// ...
public abstract class ClusterEntrypoint implements AutoCloseableAsync, FatalErrorHandler {
    public static void runClusterEntrypoint(ClusterEntrypoint clusterEntrypoint) {
        final String clusterEntrypointName = clusterEntrypoint.getClass().getSimpleName();
		try {
			clusterEntrypoint.startCluster();
		} catch (ClusterEntrypointException e) {
			LOG.error(String.format("Could not start cluster entrypoint %s.", clusterEntrypointName), e);
			System.exit(STARTUP_FAILURE_RETURN_CODE);
		}
        // ...
    }
    // ...
    public void startCluster() throws ClusterEntrypointException {
    	LOG.info("Starting {}.", getClass().getSimpleName());
        try {
            // configure default filesystem
			configureFileSystems(configuration);
            // install security context
            SecurityContext securityContext = installSecurityContext(configuration);
			securityContext.runSecured((Callable<Void>) () -> {
                // run the cluster
				runCluster(configuration);
                return null;
			});
		}
    }
}
```



- Here is the logs for default filesystem configuration.

```reStructuredText
2020-05-03 22:48:42,409 INFO  org.apache.flink.runtime.entrypoint.ClusterEntrypoint         - Install default filesystem.
2020-05-03 22:48:42,465 INFO  org.apache.flink.core.fs.FileSystem  - Hadoop is not in the classpath/dependencies. The extended set of supported File Systems via Hadoop is not available.
```



- Logs for security context

```reStructuredText
2020-05-03 22:48:42,497 INFO  org.apache.flink.runtime.entrypoint.ClusterEntrypoint         - Install security context.
2020-05-03 22:48:42,511 INFO  org.apache.flink.runtime.security.modules.HadoopModuleFactory  - Cannot create Hadoop Security Module because Hadoop cannot be found in the Classpath.

# Jaas = Java Authentication and Authorization Service(JAAS) for authentication of users and for authorization of users
2020-05-03 22:48:42,525 INFO  org.apache.flink.runtime.security.modules.JaasModule          - Jaas file will be created as /tmp/jaas-8543618536163139636.conf.
2020-05-03 22:48:42,534 INFO  org.apache.flink.runtime.security.SecurityUtils               - Cannot install HadoopSecurityContext because Hadoop cannot be found in the Classpath.
```

<br />

<br />

- After setup, cluster service will be initialized

### How cluster services are initialized?

```java
public abstract class ClusterEntrypoint implements AutoCloseableAsync, FatalErrorHandler {
	// ...
    protected void initializeServices(Configuration configuration) throws Exception {
        LOG.info("Initializing cluster services.");

		synchronized (lock) {
            // get job manager rpc address and port-range
			final String bindAddress = configuration.getString(JobManagerOptions.ADDRESS);
			final String portRange = getRPCPortRange(configuration);
			
            // create the rpc server with AkkaRpcServiceUtils
			commonRpcService = createRpcService(configuration, bindAddress, portRange);
			configuration.setString(JobManagerOptions.ADDRESS, commonRpcService.getAddress());
			configuration.setInteger(JobManagerOptions.PORT, commonRpcService.getPort());


			ioExecutor = Executors.newFixedThreadPool(
				Hardware.getNumberCPUCores(),
				new ExecutorThreadFactory("cluster-io"));
			haServices = createHaServices(configuration, ioExecutor);
            // create blobServer and start 
            // BlobServer is responsible for listening incoming and to handle them
            // BlobServer is also takes care of creating directory to store BLOBs object
			blobServer = new BlobServer(configuration, haServices.createBlobStore());
			blobServer.start();
            // create HeartBeatServices
            // HeartBeatServices gives access to all services needed for heartbeating
			heartbeatServices = createHeartbeatServices(configuration);
            // metrictregisty keeps track of all registered metrics
			metricRegistry = createMetricRegistry(configuration);

			final RpcService metricQueryServiceRpcService = MetricUtils.startMetricsRpcService(configuration, bindAddress);
			metricRegistry.startQueryService(metricQueryServiceRpcService, null);

			final String hostname = RpcUtils.getHostname(commonRpcService);

			processMetricGroup = MetricUtils.instantiateProcessMetricGroup(
				metricRegistry,
				hostname,
				ConfigurationUtils.getSystemResourceMetricsProbingInterval(configuration));

			archivedExecutionGraphStore = createSerializableExecutionGraphStore(configuration, commonRpcService.getScheduledExecutor());
		}
    }
}
```

- You will see the logs for these services.

```reStructuredText
# ...
2020-05-03 22:48:43,641 INFO  org.apache.flink.runtime.blob.BlobServer                      - Created BLOB server storage directory /tmp/blobStore-57de0a06-e2cc-481b-b0b5-26970fb77e53
2020-05-03 22:48:43,644 INFO  org.apache.flink.runtime.blob.BlobServer                      - Started BLOB server at 0.0.0.0:46005 - max concurrent requests: 50 - max backlog: 1000
# ...
```

- Here is the important services logs that's show TaskManagers register itself to the ResouceManager

```reStructuredText
2020-05-03 22:48:46,775 INFO  org.apache.flink.runtime.resourcemanager.StandaloneResourceManager  - Registering TaskManager with ResourceID 2c1556deb41732fd9124989609914a5c (akka.tcp://flink@127.0.1.1:38023/user/taskmanager_0) at ResourceManager
2020-05-03 22:48:46,833 INFO  org.apache.flink.runtime.resourcemanager.StandaloneResourceManager  - Registering TaskManager with ResourceID 2c1556deb41732fd9124989609914a5c (akka.tcp://flink@127.0.1.1:38023/user/taskmanager_0) at ResourceManager
```

<br />

<br />

## TaskManager's Log File

- These logs are similar to Job Manager's log file.

```reStructuredText
2020-05-03 22:48:43,894 INFO  org.apache.flink.runtime.taskexecutor.TaskManagerRunner       - 


TM_RESOURCES_JVM_PARAMS extraction logs:
 - Loading configuration property: jobmanager.rpc.address, localhost
 - Loading configuration property: jobmanager.rpc.port, 6123
 - Loading configuration property: jobmanager.heap.size, 1024m
 - Loading configuration property: taskmanager.memory.process.size, 1568m
 - Loading configuration property: taskmanager.numberOfTaskSlots, 8
 # ...
 2020-05-03 22:48:43,894 INFO  org.apache.flink.runtime.taskexecutor.TaskManagerRunner       -  Starting TaskManager (Version: 1.10.0, Rev:aa4eb8f, Date:07.02.2020 @ 19:18:19 CET)
2020-05-03 22:48:43,894 INFO  org.apache.flink.runtime.taskexecutor.TaskManagerRunner       -  OS current user: mehmetozanguven
2020-05-03 22:48:43,895 INFO  org.apache.flink.runtime.taskexecutor.TaskManagerRunner       -  Current Hadoop/Kerberos user: <no hadoop dependency found>
2020-05-03 22:48:43,895 INFO  org.apache.flink.runtime.taskexecutor.TaskManagerRunner       -  JVM: OpenJDK 64-Bit Server VM - AdoptOpenJDK - 11/11.0.6+10
2020-05-03 22:48:43,895 INFO  org.apache.flink.runtime.taskexecutor.TaskManagerRunner       -  Maximum heap size: 512 MiBytes
# ...
```

<br />

<br />

## What happens when you submit your job?

- Right not, let's look at the log when you submit(run) your flink job.

> Note: State Example job name was the "Flink Streaming Java API Skeleton"

- JobId is generated by Flink (JobManager's log file):

```reStructuredText
2020-05-03 22:48:57,513 INFO  org.apache.flink.runtime.dispatcher.StandaloneDispatcher      - Received JobGraph submission 35d029eee2a592940a335c8a4bfc7060 (Flink Streaming Java API Skeleton).
2020-05-03 22:48:57,514 INFO  org.apache.flink.runtime.dispatcher.StandaloneDispatcher      - Submitting job 35d029eee2a592940a335c8a4bfc7060 (Flink Streaming Java API Skeleton).
```

- Initialize job and print the restart strategy (JobManager's log file):

```reStructuredText
2020-05-03 22:48:57,538 INFO  org.apache.flink.runtime.jobmaster.JobMaster                  - Initializing job Flink Streaming Java API Skeleton (35d029eee2a592940a335c8a4bfc7060).
2020-05-03 22:48:57,549 INFO  org.apache.flink.runtime.jobmaster.JobMaster                  - Using restart back off time strategy NoRestartBackoffTimeStrategy for Flink Streaming Java API Skeleton (35d029eee2a592940a335c8a4bfc7060).
```

- Our job is start to execute (JobManager's log file):

```reStructuredText
2020-05-10 19:48:34,216 INFO  org.apache.flink.runtime.jobmaster.JobMaster                  - Starting execution of job Flink Streaming Java API Skeleton (9d9ced6c7ac0a516b1958073f0edd4e4) under job master id 00000000000000000000000000000000.
```

- **Custom Source File** has 1 parallelism (defined by Flink) (JobManager's log file):

> 9d9ced6c7ac0a516b1958073f0edd4e4 = jobId
>
> c2bff0bee2ef084a2bc6b112cce92dc3 = custom source id

```reStructuredText
2020-05-10 19:48:34,225 INFO  org.apache.flink.runtime.executiongraph.ExecutionGraph        - Job Flink Streaming Java API Skeleton (9d9ced6c7ac0a516b1958073f0edd4e4) switched from state CREATED to RUNNING.
2020-05-10 19:48:34,237 INFO  org.apache.flink.runtime.executiongraph.ExecutionGraph        - Source: Custom File Source (1/1) (c2bff0bee2ef084a2bc6b112cce92dc3) switched from CREATED to SCHEDULED.
```

- Because we have 8 parallelism, there will be eight instances with id numbers (JobManager's log file):

```reStructuredText
2020-05-10 19:48:34,237 INFO  org.apache.flink.runtime.executiongraph.ExecutionGraph        - IBBDataSource -> MapTxtToObject (1/8) (7778c56069046c5a1f4730c468a50440) switched from CREATED to SCHEDULED.
2020-05-10 19:48:34,237 INFO  org.apache.flink.runtime.executiongraph.ExecutionGraph        - IBBDataSource -> MapTxtToObject (2/8) (b73c8f013fa5d37b1e9fffffcc472768) switched from CREATED to SCHEDULED.
2020-05-10 19:48:34,237 INFO  org.apache.flink.runtime.executiongraph.ExecutionGraph        - IBBDataSource -> MapTxtToObject (3/8) (34cf48e24141ae5e811fbcf0ac6ebb1f) switched from CREATED to SCHEDULED.
2020-05-10 19:48:34,238 INFO  org.apache.flink.runtime.executiongraph.ExecutionGraph        - IBBDataSource -> MapTxtToObject (4/8) (c75e38d97a8c26b9b1ae6474470dedf0) switched from CREATED to SCHEDULED.
2020-05-10 19:48:34,238 INFO  org.apache.flink.runtime.executiongraph.ExecutionGraph        - IBBDataSource -> MapTxtToObject (5/8) (105f2dcefab3bbc45fcef49e871b298a) switched from CREATED to SCHEDULED.
2020-05-10 19:48:34,238 INFO  org.apache.flink.runtime.executiongraph.ExecutionGraph        - IBBDataSource -> MapTxtToObject (6/8) (0da90e4375dceebdd142df352e07d33e) switched from CREATED to SCHEDULED.
2020-05-10 19:48:34,238 INFO  org.apache.flink.runtime.executiongraph.ExecutionGraph        - IBBDataSource -> MapTxtToObject (7/8) (479fb6c575b6ff64f8b94c34401e9c83) switched from CREATED to SCHEDULED.
2020-05-10 19:48:34,238 INFO  org.apache.flink.runtime.executiongraph.ExecutionGraph        - IBBDataSource -> MapTxtToObject (8/8) (ac8d287bbc3bb87f59342c47b32b6373) switched from CREATED to SCHEDULED.
```

- And also we have 8 instances of (with id numbers) (JobManager's log file):

```reStructuredText
2020-05-10 19:48:34,238 INFO  org.apache.flink.runtime.executiongraph.ExecutionGraph        - FindFastestVehicle -> Sink: PrintResult (1/8) (54f54824dab58e882c20808b485112e3) switched from CREATED to SCHEDULED.
2020-05-10 19:48:34,238 INFO  org.apache.flink.runtime.executiongraph.ExecutionGraph        - FindFastestVehicle -> Sink: PrintResult (2/8) (738207a12cec4c57490d1b55290ea34b) switched from CREATED to SCHEDULED.
2020-05-10 19:48:34,238 INFO  org.apache.flink.runtime.executiongraph.ExecutionGraph        - FindFastestVehicle -> Sink: PrintResult (3/8) (6eea23540e58d5aff098a2a96db89b36) switched from CREATED to SCHEDULED.
2020-05-10 19:48:34,238 INFO  org.apache.flink.runtime.executiongraph.ExecutionGraph        - FindFastestVehicle -> Sink: PrintResult (4/8) (92576a02091d413adac412f1521e74ef) switched from CREATED to SCHEDULED.
2020-05-10 19:48:34,238 INFO  org.apache.flink.runtime.executiongraph.ExecutionGraph        - FindFastestVehicle -> Sink: PrintResult (5/8) (25e0f9a5002736cfe6f5f9ebfed1f069) switched from CREATED to SCHEDULED.
2020-05-10 19:48:34,238 INFO  org.apache.flink.runtime.executiongraph.ExecutionGraph        - FindFastestVehicle -> Sink: PrintResult (6/8) (6babf57f8c236df78d01ece66d771f63) switched from CREATED to SCHEDULED.
2020-05-10 19:48:34,238 INFO  org.apache.flink.runtime.executiongraph.ExecutionGraph        - FindFastestVehicle -> Sink: PrintResult (7/8) (2a1909729e4c10ada074ae2a18c68fc7) switched from CREATED to SCHEDULED.
2020-05-10 19:48:34,239 INFO  org.apache.flink.runtime.executiongraph.ExecutionGraph        - FindFastestVehicle -> Sink: PrintResult (8/8) (5af64e90fc740665e678b5b53f9d3853) switched from CREATED to SCHEDULED.
```

- Now JobManager connects ResourceManager (JobManager's log file):

```reStructuredText
2020-05-10 19:48:34,273 INFO  org.apache.flink.runtime.jobmaster.JobMaster                  - Connecting to ResourceManager akka.tcp://flink@localhost:6123/user/resourcemanager(00000000000000000000000000000000)
```

- After all, JobManager successfully is registered at ResourceManager and request slots (JobManager's log file):

```reStructuredText
00000000000000000000000000000000@akka.tcp://flink@localhost:6123/user/jobmanager_0 for job 9d9ced6c7ac0a516b1958073f0edd4e4. (job id)

2020-05-10 19:48:34,287 INFO  org.apache.flink.runtime.jobmaster.JobMaster                  - JobManager successfully registered at ResourceManager, leader id: 00000000000000000000000000000000.
# ...
2020-05-10 19:48:34,390 INFO  org.apache.flink.runtime.jobmaster.slotpool.SlotPoolImpl      - Requesting new slot [SlotRequestId{886601088399e7df846d7d1383d736a2}] and profile ResourceProfile{UNKNOWN} from resource manager.

2020-05-10 19:48:34,391 INFO  org.apache.flink.runtime.resourcemanager.StandaloneResourceManager  - Request slot with profile ResourceProfile{UNKNOWN} for job 9d9ced6c7ac0a516b1958073f0edd4e4 with allocation id 0c281e95bf020f4088b3c54b47ea3c9d.
# ...
```

- TaskManager offers its slot to the JobManager (TaskManager's log file):

```reStructuredText
2020-05-10 19:48:34,370 INFO  org.apache.flink.runtime.taskexecutor.JobLeaderService        - Successful registration at job manager akka.tcp://flink@localhost:6123/user/jobmanager_0 for job 9d9ced6c7ac0a516b1958073f0edd4e4.

2020-05-10 19:48:34,371 INFO  org.apache.flink.runtime.taskexecutor.TaskExecutor            - Establish JobManager connection for job 9d9ced6c7ac0a516b1958073f0edd4e4.

2020-05-10 19:48:34,375 INFO  org.apache.flink.runtime.taskexecutor.TaskExecutor            - Offer reserved slots to the leader of job 9d9ced6c7ac0a516b1958073f0edd4e4.

2020-05-10 19:48:34,401 INFO  org.apache.flink.runtime.taskexecutor.TaskExecutor            - Receive slot request 0c281e95bf020f4088b3c54b47ea3c9d for job 9d9ced6c7ac0a516b1958073f0edd4e4 from resource manager with leader id 00000000000000000000000000000000.

2020-05-10 19:48:34,402 INFO  org.apache.flink.runtime.taskexecutor.TaskExecutor            - Allocated slot for 0c281e95bf020f4088b3c54b47ea3c9d.
2020-05-10 19:48:34,402 INFO  org.apache.flink.runtime.taskexecutor.TaskExecutor            - Offer reserved slots to the leader of job 9d9ced6c7ac0a516b1958073f0edd4e4.
```



- JobManager receives slots from TaskManagers, and start to deploy the job (JobManager's log file):

```reStructuredText
2020-05-10 19:48:34,474 INFO  org.apache.flink.runtime.executiongraph.ExecutionGraph        - Deploying Source: Custom File Source (1/1) (attempt #0) to 75eef81d20fd250ba7860a5e6a4c2153 @ mehmetozanguven-ABRA-A5-V5 (dataPort=40679)

2020-05-10 19:48:34,483 INFO  org.apache.flink.runtime.executiongraph.ExecutionGraph        - IBBDataSource -> MapTxtToObject (1/8) (7778c56069046c5a1f4730c468a50440) switched from SCHEDULED to DEPLOYING.

2020-05-10 19:48:34,485 INFO  org.apache.flink.runtime.executiongraph.ExecutionGraph        - Deploying IBBDataSource -> MapTxtToObject (1/8) (attempt #0) to 75eef81d20fd250ba7860a5e6a4c2153 @ mehmetozanguven-ABRA-A5-V5 (dataPort=40679)
# ...
```



- After deploying, instances start to run  (JobManager's log file):

```reStructuredText
# ...
2020-05-10 19:48:34,756 INFO  org.apache.flink.runtime.executiongraph.ExecutionGraph        - FindFastestVehicle -> Sink: PrintResult (3/8) (6eea23540e58d5aff098a2a96db89b36) switched from DEPLOYING to RUNNING.

2020-05-10 19:48:34,760 INFO  org.apache.flink.runtime.executiongraph.ExecutionGraph        - Source: Custom File Source (1/1) (c2bff0bee2ef084a2bc6b112cce92dc3) switched from DEPLOYING to RUNNING.

2020-05-10 19:48:34,773 INFO  org.apache.flink.runtime.executiongraph.ExecutionGraph        - FindFastestVehicle -> Sink: PrintResult (5/8) (25e0f9a5002736cfe6f5f9ebfed1f069) switched from DEPLOYING to RUNNING.

2020-05-10 19:48:34,780 INFO  org.apache.flink.runtime.executiongraph.ExecutionGraph        - FindFastestVehicle -> Sink: PrintResult (2/8) (738207a12cec4c57490d1b55290ea34b) switched from DEPLOYING to RUNNING.
# ...
```



- In the meantime, we see that TaskManager also says "instances are running" (TaskManager's log file):

```reStructuredText
# ...
2020-05-10 19:48:34,741 INFO  org.apache.flink.runtime.taskmanager.Task                     - FindFastestVehicle -> Sink: PrintResult (3/8) (6eea23540e58d5aff098a2a96db89b36) switched from DEPLOYING to RUNNING.

2020-05-10 19:48:34,741 INFO  org.apache.flink.runtime.taskmanager.Task                     - Source: Custom File Source (1/1) (c2bff0bee2ef084a2bc6b112cce92dc3) switched from DEPLOYING to RUNNING.

2020-05-10 19:48:34,765 INFO  org.apache.flink.runtime.taskmanager.Task                     - FindFastestVehicle -> Sink: PrintResult (5/8) (25e0f9a5002736cfe6f5f9ebfed1f069) switched from DEPLOYING to RUNNING.

2020-05-10 19:48:34,770 INFO  org.apache.flink.runtime.taskmanager.Task                     - FindFastestVehicle -> Sink: PrintResult (2/8) (738207a12cec4c57490d1b55290ea34b) switched from DEPLOYING to RUNNING.
# ...
```



- After reading `.txt` file, State changed to Finished (JobManager's log file):

```reStructuredText
# ...
2020-05-10 19:48:40,948 INFO  org.apache.flink.runtime.executiongraph.ExecutionGraph        - FindFastestVehicle -> Sink: PrintResult (2/8) (738207a12cec4c57490d1b55290ea34b) switched from RUNNING to FINISHED.

2020-05-10 19:48:40,952 INFO  org.apache.flink.runtime.executiongraph.ExecutionGraph        - FindFastestVehicle -> Sink: PrintResult (3/8) (6eea23540e58d5aff098a2a96db89b36) switched from RUNNING to FINISHED.

2020-05-10 19:48:40,956 INFO  org.apache.flink.runtime.executiongraph.ExecutionGraph        - FindFastestVehicle -> Sink: PrintResult (5/8) (25e0f9a5002736cfe6f5f9ebfed1f069) switched from RUNNING to FINISHED.

2020-05-10 19:48:41,141 INFO  org.apache.flink.runtime.executiongraph.ExecutionGraph        - Job Flink Streaming Java API Skeleton (9d9ced6c7ac0a516b1958073f0edd4e4) switched from state RUNNING to FINISHED.
# ...
```



- In the meantime, here is the TaskManager (TaskManager's log file):

```reStructuredText
# ...
2020-05-10 19:48:40,942 INFO  org.apache.flink.runtime.taskmanager.Task                     - FindFastestVehicle -> Sink: PrintResult (2/8) (738207a12cec4c57490d1b55290ea34b) switched from RUNNING to FINISHED.

2020-05-10 19:48:40,942 INFO  org.apache.flink.runtime.taskmanager.Task                     - Freeing task resources for FindFastestVehicle -> Sink: PrintResult (2/8) (738207a12cec4c57490d1b55290ea34b).

2020-05-10 19:48:40,942 INFO  org.apache.flink.runtime.taskmanager.Task                     - Ensuring all FileSystem streams are closed for task FindFastestVehicle -> Sink: PrintResult (2/8) (738207a12cec4c57490d1b55290ea34b) [FINISHED]

2020-05-10 19:48:40,942 INFO  org.apache.flink.runtime.taskexecutor.TaskExecutor            - Un-registering task and sending final execution state FINISHED to JobManager for task FindFastestVehicle -> Sink: PrintResult (2/8) 738207a12cec4c57490d1b55290ea34b.
# ...
```



- Finally, connection between (JobManager & ResourceManager), (JobManager & TaskManager) closed (JobManager's log file):

````reStructuredText
2020-05-10 19:48:41,162 INFO  org.apache.flink.runtime.jobmaster.JobMaster                  - Close ResourceManager connection d6d73032757e1a231274935bddee5e9f: JobManager is shutting down..
````

-  (TaskManager's log file)

```reStructuredText
2020-05-10 19:48:41,183 INFO  org.apache.flink.runtime.taskexecutor.TaskExecutor            - Close JobManager connection for job 9d9ced6c7ac0a516b1958073f0edd4e4.

2020-05-10 19:48:41,187 INFO  org.apache.flink.runtime.taskexecutor.TaskExecutor            - Close JobManager connection for job 9d9ced6c7ac0a516b1958073f0edd4e4.
```



<br />

Last but not least, wait for the next post...