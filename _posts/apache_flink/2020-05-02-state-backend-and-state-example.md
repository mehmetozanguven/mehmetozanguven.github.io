---
layout: post
title:  "Apache Flink Series 8 - State Backend & State Example"
date:   2020-05-02 14:30:31 +0530
categories: "apache-flink"
author: "mehmetozanguven"
---

In this post, I am going to explain “what is state backend”, “which options do we have for state backend” , “how to configure state backend for your flink job” and “I am going to implement a project to show each state backend options”.

> You can find the example from my [github](https://github.com/mehmetozanguven/flink_examples/tree/master/StateExample) repo.

Before starting, it is a good to remember what is a state in Flink:

> At a high level, we can consider state as memory in operators in Flink that remembers information about past input and can be used to influence the processing of future input.

## What is the State Backend

State backend is a pluggable component which determines how the state is stored, accessed and maintained. Because it is pluggable, two flink applications can use different state backend mechanism.

State backend is responsible for two things:

1. Local State management
2. Checkpointing state to a remote location

For local state management:

- State backend stores all keyed states and ensures that all states can be accessible correctly to the current key.

For state checkpointing:

- Because TaskManager may fail at any point in time, its values can be considered as volatile. A state backend takes care of checkpointing the state of a task to the remote and persistent location. The remote location could be a distributed file system or a database system.

In general, with State backend mechanism, we can determine the saving folder for job’s state, types of the state backend and more. 

In Flink, we have 4 options for state backend:

1. The MemoryStateBackend
2. The FsStateBackend
3. The RocksDBStateBackend
4. Custom State Backend mechanism (I am not touching this one!!)

Let’s talk about each of them one by one.



### The MemoryStateBackend (enabled by default)

- Holds data internally as objects on the Java heap. Each TaskManager process stores state in the its heap.
- When a checkpoint is taken, MemoryStateBackend sends the state to the JobManager, which stores it in its heap memory. Therefore there should be enough memory in the JobManager process to hold the all states.
- Even this approach provides very low latency to read or write state, it has some drawbacks. First, if the state of a task instance grows too large, the JVM and all task instances running on it can be killed due to an `OutOfMemoryError` . Secondly, because the states on the heap are long-lived object, this can issue on the garbage collection. Lastly, because all states store in the JobManager’s heap, if JobManager fails, all states will be lost.
- In general MemeoryStateBackend is only recommended for development environment.

### The FsStateBackend

- This approach stores the local state on the TaskManager’s JVM heap, like the MemoryStateBackend.
- However instead of checkpointing the state to the JobManager’s volatile memory, FsStateBackend writes the state to the remote file location or persistent database system.
- This approach can suffer also from `OutOfMemoryError` , because the states are held on the heap.

### The RocksDBStateBackend

> Note: RocksDB is an embeddable persistent key-value store for fast storage. More on this [**address**](https://rocksdb.org/)

- RocksDBStateBackend stores all state into local RocksDB instances.
- In order to read and write data from RockDB, it needs to be de/serialized.
- This approach can be written into remote and persistent file system when checkpointing is enabled.
- Because rocksdb writes data to disk and supports incremental checkpoints, this approach is a good choice for applications with very large state.

<br />

<br />

## State Types in Flink

- There are two types of state in Flink: **Keyed State** & **Operator State** and each of them has two forms called **Managed State** & **Raw State**.

### Operator State

- Operator state is scoped to an operator task.
- All records processed by the same parallel task have access to the same state. Don't think that all tasks are accessing the same state storage. Operator state can not be accessed by another task of the same or different operator.
- Operator state can only be implemented by implement `ListCheckpointed` interface.

<img src="/assets/apache_flink/operator-state.png" alt="operator-state.png" title="Operator State" />

- Flink offers three primitives for operator state:
  - **List State** : Represents state as a list of entries
  - **Union List State**: Represents state as a list of entries as well. But it differs from regular list state in how it is restored in the case of a failure.
  - **Broadcast state**: Designed for the special case where the state of each task of an operator is identical

### Keyed State

- It is maintained and accessed with respect to a key. For example if a key is "Tiger" then you will get state value(s) for a Tiger key, or if a key is "Lion", get state value(s) for a Lion key etc ..

<img src="/assets/apache_flink/keyed-state.png" alt="keyed-state.png" title="Keyed State" />

- Here is the available state primitives for KeyedState (detail information can be found in this [link](https://ci.apache.org/projects/flink/flink-docs-stable/dev/stream/state/state.html#using-managed-keyed-state):
  - `ValueState<T>` : keeps only one value and this value can be updated via `update(T)`
  - `ListState<T>` : keeps a list of elements for specific key
  - `ReducingState<T>` : keeps single value that represents the aggregation of all values added to the state.
  - `Aggregating<IN, OUT>` : Same features with reducing state, only input type can be different.
  - `MapState<UK, UV>` : Keeps a list of mappings.

### Managed State

- Managed State is represented in data structures controlled by the Flink runtime. Examples: `ValuState, ListState` etc..
- Flink's runtime encodes the states and writes them into the checkpoints

### Raw State

- It is a state which has own data structures. Flink does not anything about these kind of states. Flink only write a sequence of bytes into the checkpoint. Raw state can be used when you are implementing customized operators.

<br />

<br />

## State Example 

Let’s create a flink example to see different state backend options more realistic way. 

In this example, I going to use big csv file provided by [https://data.ibb.gov.tr](https://data.ibb.gov.tr/) and I am going to use the specific data set called [***INSTANT LOCATION AND SPEED INFORMATION OF IMM ISTAC VEHICLES\***](https://data.ibb.gov.tr/en/dataset/ibb-istac-araclarinin-anlik-konum-ve-hiz-bilgileri) ***(\***The dataset contains the location and speed information of trucks, vans, road sweepers, and road washing vehicles of ISTAC Inc., a subsidiary of the Istanbul Metropolitan Municipality.***).\***

This data set are divided by ten days for each month. I downloaded one part of the data set. However one part was too big to read on my local machine. Then I decided to the split data with pandas and then convert them to the txt file to read from Flink application. 

> Note: Flink may also be providing classes to transform csv file to datastream, but pandas way was more easy to do for me!!

Here is the python code for splitting the csv file (you may change split number) and converting to the txt:

```python
import pandas as pandas
# read csv file and convert them to .txt
path = "path/To/CSV"
csv = pandas.read_csv(path)
range = csv[:500000] # take only first 500000 rows
file = open("path/To/Txt", "w+")
for each in range.itertuples():
    str_repr = str(each[1]) + "," + str(each[2]) \
               + "," + str(each[3]) + "," + str(each[4]) + "," + str(each[5]) \
               + "," + str(each[6]) + "," + str(each[7]) + "," + str(each[8])\
               + "," + str(each[9]) + "," + str(each[10])
    file.write(str_repr+"\n")
file.close()
```

<br />

<br />

### Our Flink Job

In this example, our flink job will find the "fastest vehicle" for each type in a real-time way. When we are finding the fastest vehicle, we are going to use `ValueState` (which is Managed KeyedState) and `MemoryStateBackend,  FsStateBackend and RocksDbStateBackend` respectively.

Basically our flink application:

- will group the all different vehicle type (via KeyedStream)
- then, we will apply map function for the specific type of vehicle
- for each map function, we will get the previous recorded vehicle for that vehicle type(key)
- If previous record's speed is lower than the current one, we will update the state.
- After all we will print the fastest vehicles.

First let's summarize how we define Managed KeyedState for our Flink Job:

#### How do you define Managed KeyedState for your Flink Job?

1. Decide which state primitives you want to use. (Could it be ValueState or ListState or others ?)
2. State is accessed via `RuntimeContext` , therefore your function must extends `RichFunction` 
3. Define correspond `StateDescriptor` to get a state value. For example if you decided to use `ListState`, you would define `ListStateDescriptor`. 
4. Access your state, within an `void open(Configuration config) ` function.
5. After all, you can get state's value via appropriate method call such as `.getAll(), .value()`  etc.. 

<br />

<br />

#### Let's Build Flink Application

- Create simple flink application via its maven archetype:

```bash
$ mvn archetype:generate                               \
      -DarchetypeGroupId=org.apache.flink              \
      -DarchetypeArtifactId=flink-quickstart-java      \
      -DarchetypeVersion=1.10.0
```



- After setup, I will need a class for converting the file content to the correspond object. To do that first I created a class called `VehicleInstantData`which contains all the csv’s fields as attributes (with format String).

```java
public class VehicleInstantData implements Serializable
{
    private String _id;
    private String day_hour;
    private String geohash;
    private String latitude;
    private String longitude;
    private String vehicle_type;
    private String speed;
    private String day_year;
    private String day_mounth;
    private String day_day;
    
    public static VehicleInstantData createFromSplittedArray(String[] splittedArray)
    {
        VehicleInstantData newData = new VehicleInstantData(
                splittedArray[0], splittedArray[1], splittedArray[2],
                splittedArray[3], splittedArray[4], splittedArray[5],
                splittedArray[6], splittedArray[7], splittedArray[8],
                splittedArray[9]);

        return newData;
    }

  	private VehicleInstantData(String _id, String day_hour, String geohash, 
                               String latitude, String longitude, 
                               String vehicle_type, String speed, 
                               String day_year, String day_mounth, String day_day) {
        this._id = _id;
        this.day_hour = day_hour;
        this.geohash = geohash;
        this.latitude = latitude;
        this.longitude = longitude;
        this.vehicle_type = vehicle_type;
        this.speed = speed;
        this.day_year = day_year;
        this.day_mounth = day_mounth;
        this.day_day = day_day;
    }
// empty constructor
// getter and setter
}
```



- To convert string file content to correspond object, I used map transformation

````java
public class MapToObjectTransformation implements MapFunction<String, VehicleInstantData>
{

    @Override
    public VehicleInstantData map(String value) throws Exception
    {
        String[] splittedByComma = value.split(",");
        return VehicleInstantData.createFromSplittedArray(splittedByComma);
    }
}
// ...

public class StreamCreator
{
    public DataStream<String> getDataSourceStream(StreamExecutionEnvironment env, String filePath)
    {
        return env.readTextFile(filePath);
    }

    public DataStream<VehicleInstantData> mapDataSourceStreamToObject(DataStream<String> dataSourceStream)
    {
        return dataSourceStream.map(new MapToObjectTransformation());
    }
}
````



- Now, I grouped the stream flow by vehicle types (I created a KeyedStream by VehicleTypes)

````java
public class StreamCreator
{
    // ...
    public KeyedStream<VehicleInstantData, Tuple> keyByVehicleType(DataStream<VehicleInstantData> vehicleStream)
    {
        return vehicleStream
                .keyBy("vehicle_type");
    }
    // ...
}

// ...
public class StreamingJob {

	public static void main(String[] args) throws Exception {
		// ...
		KeyedStream<VehicleInstantData, Tuple> keyByVehicleType = streamCreator.keyByVehicleType(vehicleInstantDataDataStream);
		// ...

		// execute program
		env.execute("Flink Streaming Java API Skeleton");
	}
}
````



- After KeyedStream, I am going to map the fastest vehicle as a **SingleOutputStreamOperator**.

```java
public class StreamCreator
{
    // ...
    public SingleOutputStreamOperator<VehicleInstantData> findFastestVehicleForEachType(KeyedStream<VehicleInstantData, Tuple> vehicleKeyedStreamByVehicleType)
    {
        return vehicleKeyedStreamByVehicleType.map(new FastestVehicleMapper());
    }
}
// ...
public class FastestVehicleMapper extends RichMapFunction<VehicleInstantData, VehicleInstantData>
{
    // ValueState only holds one object for each key
    private transient ValueState<VehicleInstantData> fastestVehicleState;

    @Override
    public void open(Configuration parameters) throws Exception
    {
        ValueStateDescriptor<VehicleInstantData> valueStateDescriptor = new ValueStateDescriptor<VehicleInstantData>(
                "fastest-vehicle",
                TypeInformation.of(VehicleInstantData.class)
        );
        
        fastestVehicleState = getRuntimeContext().getState(valueStateDescriptor);
    }

    @Override
    public VehicleInstantData map(VehicleInstantData newVehicleData) throws Exception
    {
        VehicleInstantData previousVehicleData = this.fastestVehicleState.value();
        
        if (previousVehicleData == null) // means there is nothing in the state
        {
            this.fastestVehicleState.update(newVehicleData);
            return newVehicleData;
        }
        else
        {
            return updateStateAndReturnValue(newVehicleData, previousVehicleData);
        }
    }

    private VehicleInstantData updateStateAndReturnValue(VehicleInstantData newVehicleData, VehicleInstantData previousVehicleData) throws IOException 
    {
        if (Integer.parseInt(previousVehicleData.getSpeed()) < Integer.parseInt(newVehicleData.getSpeed()))
        {
            // update the state with new one
            this.fastestVehicleState.update(newVehicleData);
            return newVehicleData;
        }
        else
        {
            return previousVehicleData;
        }
    }
}
```



- Finally, my main method:

```java
public class StreamingJob {

	public static void main(String[] args) throws Exception {
		String txtFilePath = "/home/mehmetozanguven/Desktop/flink_examples/datas/ibb_datas/01.2019_1-10.txt";
		// set up the streaming execution environment
		final StreamExecutionEnvironment env = StreamExecutionEnvironment.createLocalEnvironment();

		StreamCreator streamCreator = new StreamCreator();

		DataStream<String> dataSource = streamCreator.getDataSourceStream(env, txtFilePath);

		DataStream<VehicleInstantData> vehicleInstantDataDataStream = streamCreator.mapDataSourceStreamToObject(dataSource);

		KeyedStream<VehicleInstantData, Tuple> keyByVehicleType = streamCreator.keyByVehicleType(vehicleInstantDataDataStream);

		SingleOutputStreamOperator<VehicleInstantData> fastestVehicles = streamCreator.findFastestVehicleForEachType(keyByVehicleType);

		fastestVehicles.print();

		// execute program
		env.execute("Flink Streaming Java API Skeleton");
	}
}
```



- Now, our flink application is ready, take a jar your application via `mvn clean install` 

- Let's configure state backend

  

<br />

<br />

### General Configuration for State Backend

Even if you are using `MemoyStateBackend` for state backend, you should configure the savepoints and checkpoints directory in the **flink-conf.yaml** file. These directories will play in role when you want to save your all state in a current time or periodically.

> I intent to write a post for savepoint & checkpoint mechanism after State backend with the same example as we are doing right now.

Here is the config for savepoints and checkpoints:

```yaml
# ...
state.checkpoints.dir: file:///home/path/to/checkpoints_dir
state.savepoints.dir: file:///home/path/to/savepoints_dir
# ...
```



### Configure MemoryStateBackend for Our Flink Job

Because `MemoryStateBackend` is a default option, we don't need to setup anything. I have just deployed the Job.

Here is the log file after I deployed the jar & submit the application:

#### JobManager's Log File

- As below, Flink have seen the our savepoints & checkpoints configuration

```reStructuredText
2020-05-03 18:29:25,293 INFO  org.apache.flink.configuration.GlobalConfiguration            - Loading configuration property: parallelism.default, 1

2020-05-03 18:29:25,294 INFO  org.apache.flink.configuration.GlobalConfiguration            - Loading configuration property: state.checkpoints.dir, file:///home/mehmetozanguven/Desktop/ApacheTools/flink-1.10.0/state_backend/memory_state_backend/checkpoints_dir

2020-05-03 18:29:25,294 INFO  org.apache.flink.configuration.GlobalConfiguration            - Loading configuration property: state.savepoints.dir, file:///home/mehmetozanguven/Desktop/ApacheTools/flink-1.10.0/state_backend/memory_state_backend/savepoints_dir

2020-05-03 18:29:25,294 INFO  org.apache.flink.configuration.GlobalConfiguration            - Loading configuration property: 
```

- Here is the MemoryStateBackend configuration:

```reStructuredText
2020-05-03 18:29:37,807 INFO  org.apache.flink.runtime.jobmaster.JobMaster                  - No state backend has been configured, using default (Memory / JobManager) MemoryStateBackend (data in heap memory / checkpoints to JobManager) (checkpoints: 'file:/home/mehmetozanguven/Desktop/ApacheTools/flink-1.10.0/state_backend/memory_state_backend/checkpoints_dir', savepoints: 'file:/home/mehmetozanguven/Desktop/ApacheTools/flink-1.10.0/state_backend/memory_state_backend/savepoints_dir', asynchronous: TRUE, maxStateSize: 5242880)
```



#### TaskManager's Log File

- Here is the MemoryStateBackend configuration:

```reStructuredText
2020-05-03 18:29:38,168 INFO  org.apache.flink.streaming.runtime.tasks.StreamTask           - No state backend has been configured, using default (Memory / JobManager) MemoryStateBackend (data in heap memory / checkpoints to JobManager) (checkpoints: 'file:/home/mehmetozanguven/Desktop/ApacheTools/flink-1.10.0/state_backend/memory_state_backend/checkpoints_dir', savepoints: 'file:/home/mehmetozanguven/Desktop/ApacheTools/flink-1.10.0/state_backend/memory_state_backend/savepoints_dir', asynchronous: TRUE, maxStateSize: 5242880)

2020-05-03 18:29:38,168 INFO  org.apache.flink.streaming.runtime.tasks.StreamTask           - No state backend has been configured, using default (Memory / JobManager) MemoryStateBackend (data in heap memory / checkpoints to JobManager) (checkpoints: 'file:/home/mehmetozanguven/Desktop/ApacheTools/flink-1.10.0/state_backend/memory_state_backend/checkpoints_dir', savepoints: 'file:/home/mehmetozanguven/Desktop/ApacheTools/flink-1.10.0/state_backend/memory_state_backend/savepoints_dir', asynchronous: TRUE, maxStateSize: 5242880)

2020-05-03 18:29:38,178 INFO  org.apache.flink.runtime.taskmanager.Task                     - Split Reader: Custom File Source -> MapTxtToObject (1/1) (7956c3e739a4fd7b67cd932c27822a60) switched from DEPLOYING to RUNNING.
2020-05-03 18:29:38,179 INFO  org.apache.flink.streaming.runtime.tasks.StreamTask           - No state backend has been configured, using default (Memory / JobManager) MemoryStateBackend (data in heap memory / checkpoints to JobManager) (checkpoints: 'file:/home/mehmetozanguven/Desktop/ApacheTools/flink-1.10.0/state_backend/memory_state_backend/checkpoints_dir', savepoints: 'file:/home/mehmetozanguven/Desktop/ApacheTools/flink-1.10.0/state_backend/memory_state_backend/savepoints_dir', asynchronous: TRUE, maxStateSize: 5242880)
```

<br />

### Configure FsStateBackend for Our Flink Job

- Update `flink-conf.yaml` file like this:

````yaml
# Flink will know that we are going to use FsStateBackend
state.backend: filesystem 

state.checkpoints.dir: file:///home/mehmetozanguven/Desktop/ApacheTools/flink-1.10.0/state_backend/memory_state_backend/checkpoints_dir

state.savepoints.dir: file:///home/mehmetozanguven/Desktop/ApacheTools/flink-1.10.0/state_backend/memory_state_backend/savepoints_dir

````

<br />

### Configure RocksDbStateBackend for Our Flink Job

- Update `flink-conf.yaml` file like this:

```yaml
# Flink will know that we are going to use RocksDb
state.backend: rocksdb

state.checkpoints.dir: file:///home/mehmetozanguven/Desktop/ApacheTools/flink-1.10.0/state_backend/memory_state_backend/checkpoints_dir

state.savepoints.dir: file:///home/mehmetozanguven/Desktop/ApacheTools/flink-1.10.0/state_backend/memory_state_backend/savepoints_dir
```

- And also add the following dependency:

```xml
<dependency>
    <groupId>org.apache.flink</groupId>
    <artifactId>flink-statebackend-rocksdb_2.11</artifactId>
    <version>{flink_versiyon}</version>
    <scope>provided</scope>
</dependency>
```

<br />

<br />

In the next post, we are going to read log files for this example.

After that, I am going to explain savepoints & checkpoints for flink application.

Then, I am going to do detail post for rocksdb configuration with an example later on.

Hopefully,

<br />

<br />

Last but not least, wait for the next post …