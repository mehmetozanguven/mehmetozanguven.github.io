---
layout: post
title: "Apache Flink Series 4 — DataStream API"
date: 2020-02-18 19:45:31 +0530
categories: "apache-flink"
author: "mehmetozanguven"
newUrl: "https://mehmetozanguven.com/apache-flink/datastream-api/"
---

In this post, I am going to explain DataStream API in Flink.

> You may see the all my notes about Apache Flink with this [link](/apache-flink/)

When we look at the Flink as a software, Flink is built as layered system. And one of the layer is DataStream API which places top of Runtime Layer.

Let’s dive into DataStream API with transformations in the Flink.

## Transformations

- A stream **transformation** is applied on one or more streams and converts them into one or more output streams.
- Most stream transformation are based on user-defined functions. Functions define how the elements of the input stream are transformed into elements of the output stream.
- Most of the functions(maybe all of them, I am not sure) are designed as Single Abstract Method(SAM), therefore you can use lambda expression as well.
- We can categorized transformation to 4 sections:
  - Basic transformation
  - KeyedStream transformation
  - MultiStream transformation
  - Distribution Transformation

### Basic Transformation

- Process individual events, meaning that each output record was produced from a single input record

#### Basic Transformation — Map

- It is called with DataStream.map() and produces a new DataStream with defined function

```java
DataStream<Integer> dataStream = //... your data source kafka topic, file etc..
  //MapFunction<I,O> accepts input (which is Integer in this example),
  // and produces new datastream with the desired output(which is Integer also in this example)
dataStream.map(new MapFunction<Integer, Integer>() {
    @Override
    public Integer map(Integer value) throws Exception {
        return 2 * value; // our datasource has value of Integers, we double these integers
    }
});
```

#### Basic Transformation —Filter

- It is called with `DataStream.filter()`and produces a new DataStream of the same type
- A filter transformations drops(removed) of events of a stream by evaluating a boolean condition on each input.
- A return value true means that event will forward to the new data stream.
- A return value false means that event will drop.

```java
DataStream<LogObject> yourLogs =... // kafka topic, file etc..
  // return new stream which has no key in the log object
DataStream<LogObject>streamWithoutHavingKey = yourLogs.filter(new FilterFunction<LogObject>() {
    @Override
    public boolean filter(LogObject value) throws Exception {
        return !value.isKey();
    }
});
```

#### Basic Transformation —FlatMap

- Similar to map transformation, but it can produce zero, one or more results
- Because it may produce more results, its output type will wrap with `Collector<T>` which collects the records and forwards it.

```java
DataStream<LogObject> yourLogs =... // kafka topic, file etc..
yourLogs.flatMap(new FlatMapFunction<LogObject, String>() {
    @Override
    public void flatMap(LogObject value, Collector<String> out)
        throws Exception {
        for(String word: value.getUserName().split(" ")){
            out.collect(word);
        }
    }
});
```

### Keyed Transformation

- Most of the time you want to group your events that share a certain property together. For example, you may want to look at the “count number of invalid token for each color”. This can be done via sql like this: (let’s assume that if the token is not null, then it is valid)

```sql
SELECT tableName.color, count(token) AS countOfValidToken
FROM tableName
WHERE tableName.token IS NOT NULL
GROUP BY tableName.color;
```

- Same thing can be done in Flink via Keyed Transformation. DataStream API provides this feature called KeyedStream.
- Note: Use KeyedStream with care. If the key domain is continuously growing (you could have unique key for each event), then don’t forget to clean up state for keys that are no longer active to avoid memory problems.

#### Keyed Transformation — keyBy

- The keyBy transformation converts a DataStream into a KeyedStream by specifying a key.
- Based on the key, the same key are processed by the same task.

<img src="/assets/apache_flink/key_by_operation.png" alt="key_by_operation" title="keyBy operation" />

```java
DataStream<LogObject> yourLogs =... // kafka topic, file etc..
// Note: assumed that LogObject has attibute called color, this one way of the defining key in the Flink, we will touch this later
DataStream<LogObject, String> eachColorStream = yourLogs.keyBy(color)
```

#### Keyed Transformation — Rolling Aggregations

- Rolling aggregations are applied on a KeyedStream and produce a DataStream of aggregates.
- A rolling aggregation does not require a user-defined functions.
- Rolling aggregation methods: **sum(), min(), max(), minBy(), maxBy()**

#### Keyed Transformation — Reduce

- Combines the current element with last reduced element and emits the new value.

```java
keyedStream.reduce(new ReduceFunction<Integer>(){
    @override
    public Integer reduce(Integer value1, Integer value2){
        return value1+value2;
    }
}
```

### MultiStream Transformations

- Used to merge different streams or separate stream into different streams

#### MultiStream Transformations — Union

- Merges two or more DataStreams of the same type and produces a new DataStream of the same type.
- The events are merged in a FIFO fashion, the operator does not produce a specific order of events.
- The union operator does not perform duplication elimination.

<img src="/assets/apache_flink/union_operation.png" alt="union_operation" title="union operation" />

#### MultiStream Transformations — Split

- Split is the inverse transformation to the union transformation
- It divides an input stream into two or more output stream of the same type as the input stream.

### Distribution Transformation

- These operations define how events are assigned to tasks.
- If you don’t specify one, DataStream API automatically will choose the one strategy depending on the parallelism etc..

#### Distribution Transformation — Random

- Distributes records randomly
- Called with `dataStream.shuffle()`

#### Distribution Transformation —Round-Robin

- Distributes events in round-robin fashion
- Called with `dataStream.rebelance()`

#### Distribution Transformation — Rescale

- Distributes events in a round-robin fashion, but only to a subset of successor tasks
- Called with `dataStream.rescale()`

#### Distribution Transformation —Broadcast

- Replicates the input data stream so that all the events are sent to all parallel tasks of the downstream operator.
- Called with `dataStream.broadcast()`

<br />

## Defining Keys

- We can define keys for keyedStream in 3 ways

### 1. Field Positions

- If the data type is tuple, keys can be defined by simply using the field position of the corresponding tuple element.

```java
DataStream<Tuple3<Integer,String,Long>> input =
// defined DataStream as Tuple which has 3 objects
KeyedStream<Tuple3<Integer,String,Long>,Tuple> keyed = input.keyBy(0) // keyed stream with integer input of the tuple
```

### 2. Field Expressions

- Define keys by using String based field expressions.
- Field expressions work for tuples, POJOs and case classes

```java
public class LogObject{
    private String color;
}

DataStream<LogObject> input = // ...

KeyedStream<LogObject, String> keyed = input.keyBy("color")
```

### 3. KeySelector

- Specify keys with KeySelector functions.
- KeySelector function extracts a key from an input event.

```java
KeyedStream<LogObject, String> keyed = input.keyBy(new KeySelector<LogObject, String>(){
    public String getKey(LogObject logObject){
        return logObject.token;
    }
});
```

<br />

## Rich Functions

- We use rich functions when there is a need to initialize a function before it processes the first record or to retrieve information about the context in which it is executed.
- The name of the rich function starts with Rich followed by the transformation name RichMapFunction, RichFlatMapFunction etc…
- When we are using rich function, we have 2 additional method:
  - **open()** => is an initialization method for the rich function. It is called **once per task**.
  - **close()**=> is an finalization method. It is called **once per task** after the last call of the transformation

<br />

## Setting the Parallelism

As already know, Flink applications are executed in parallel in a distributed environment.

- Let’s remember how it happens:
  - When a DataStream program is submitted to the JobManager(could be done via Dashboard or command line), the system creates a dataflow graph and prepares the operators for execution
  - Each operator will be converted to the parallel tasks.
  - Each task will process subset of the operator’s input stream
  - The **#parallel tasks** of an operator is called **the parallelism of the operator**

Now we can control this parallelism, when we are writing DataStream program. And also we can control parallelism of the execution environment or per individual operator.

And don’t forget that by default, the parallelism of all operators is set to the parallelism of the application’s execution environment

Let’s look at this example:

```java
StreamExecutionEnvironment env = StreamExecutionEnvironment.createLocalEnvironment();

int defaultParallelism = env.getParallelism();

// the source will runs with the default parallelism
DataStream<LogObject> source = env.addSource(YourApahceFlinkKafkaConsumer);

// the map parallelism is set to double the defaul parallelism
DataStream<LogObject> mappedLog = source.map(...).setParallelism(defaultParallelism*2);

// the filterLog is fixed to 2
DataStream<LogObject> filterLog = source.filter(...).setParallelism(2);
```

If we submit this application with parallelism 16, then:

- Source will run with 16 tasks
- Mapper will run with 32 tasks
- Filter will run with 2 tasks

Last but not least, wait for the next post …
