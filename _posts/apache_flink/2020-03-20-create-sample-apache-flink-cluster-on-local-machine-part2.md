---
layout: post
title:  "Apache Flink Series 7 — Create Sample Apache Flink Cluster on Local Machine — Part 2"
date:   2020-03-20 19:30:31 +0530
categories: "apache-flink"
author: "mehmetozanguven"
---

In this post, I will create simple stream job and submit the job to the flink cluster. You can find the project in [my github repo](https://github.com/mehmetozanguven/flink_examples/tree/master/SampleStream).


This is the second part of the sub-series of my Flink posts.

When you write your stream job, you probably will follow these steps:

1. You will setup the execution environment.

2. You will write streams for your data sources (it can be kafka connector, reading file etc..)

3. You will convert the your stream(s) to another one. (mapping, filtering etc..)

4. You will write the result(which is another stream — data sink — ) to kafka, mongodb etc …

<br />

Main method at the end of the post:

```java
import com.sampleFlinkProject.steams.StreamCreator;
import org.apache.flink.api.java.tuple.Tuple2;
import org.apache.flink.streaming.api.datastream.DataStream;
import org.apache.flink.streaming.api.datastream.SingleOutputStreamOperator;
import org.apache.flink.streaming.api.environment.StreamExecutionEnvironment;

public class StreamingJob
{

	public static void main(String[] args) throws Exception
	{
		// set up the streaming execution environment
		final StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();

		StreamCreator streamCreator = new StreamCreator();

		DataStream<String> wordCountDataSource = streamCreator.createDataSourceStreamForWordCount(env);

		DataStream<Tuple2<String, Integer>> readWordByWordStream = streamCreator.splitSentenceWordByWord(wordCountDataSource);
		DataStream<Tuple2<String, Integer>> filterStreamContainsSpecificCharacter = streamCreator.filterWordThatContainsSpecificCharacter(readWordByWordStream, "a");

		SingleOutputStreamOperator<Tuple2<String, Integer>> summedStream = streamCreator.sumTheWordContainsSpecificCharacter(filterStreamContainsSpecificCharacter);
		summedStream.print();

		env.execute("Flink word count example");
	}
}
```

<br />
<br />

Before following these steps, you must create a simple starting architecture for the stream job. You may import all the maven dependencies by yourself. But Flink provides maven archetype for quick start, here is the command for that:

```bash
$ mvn archetype:generate                               \
      -DarchetypeGroupId=org.apache.flink              \
      -DarchetypeArtifactId=flink-quickstart-java      \
      -DarchetypeVersion=1.10.0
```

After typing, you are requested to groupId, artifactId and version. Then project will be created.

Here is the default view of our project:

<img src="/assets/apache_flink/default_structure_of_flink_program.png" alt="default_structure_of_flink_program.png" title="Default Structure of Flink Project" />

As you can see, **step 1(execution environment setup) has already done by Flink’s maven archetypes.**

<br />


## Step 2 (write streams for data source)

In this project, we will count the words in the arraylist (this list can be found on the [Flink’s github](https://github.com/apache/flink/blob/master/flink-examples/flink-examples-streaming/src/main/java/org/apache/flink/streaming/examples/wordcount/util/WordCountData.java)), then we will apply some transformations on it.

Here is the `WordCountData.java` :

```java
package com.sampleFlinkProject.util;

public class WordCountData
{
    public static final String[] WORDS = new String[]{
            "To be, or not to be,--that is the question:--",
            "Whether 'tis nobler in the mind to suffer",
            "The slings and arrows of outrageous fortune",
            "Or to take arms against a sea of troubles,",
            "And by opposing end them?--To die,--to sleep,--",
            "...",
            "..."
    };
}
```

I have created another class called `com.sampleFlinkProject.steams.StreamCreator` where I will create all the related streams in it. Let’s create our data source stream and put to the main entry.

```java
package com.sampleFlinkProject.steams;

import com.sampleFlinkProject.util.WordCountData;
import org.apache.flink.streaming.api.datastream.DataStream;
import org.apache.flink.streaming.api.environment.StreamExecutionEnvironment;


public class StreamCreator
{
    public DataStream<String> createDataSourceStreamForWordCount(StreamExecutionEnvironment env)
    {
        DataStream<String> source = env.fromElements(WordCountData.WORDS);
        return source;
    }
}
```

```java
package com.sampleFlinkProject;

import com.sampleFlinkProject.steams.StreamCreator;
import org.apache.flink.streaming.api.datastream.DataStream;
import org.apache.flink.streaming.api.environment.StreamExecutionEnvironment;

public class StreamingJob 
{

	public static void main(String[] args) throws Exception 
	{
		// set up the streaming execution environment
		final StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
		StreamCreator streamCreator = new StreamCreator();

		DataStream<String> wordCountDataSource = streamCreator.createDataSourceStreamForWordCount(env);

		// execute program
		env.execute("Flink word count example");
	}
}
```

<br />

## Step 3 (adding more streams)

Now, I will split the sentence word by word by and I will count the word(s) that contains `a or A`

For split operation I will use FlatMap operation, because split operation may produce zero, one or more output events.

For counting part, I will use the KeyedStream operation. First I will transform the DataStream into the KeyedStream by specifying a key (key will be something like: “Hey, if word contains a or A then this event will be processed by the same task), after that I will apply the sum() operation. (Don’t forget that aggregation functions can only be used with KeyedStream)

Here is the part for split operation:

```java
public DataStream<Tuple2<String, Integer>> splitSentenceWordByWord(DataStream<String> wordDataSourceStream)
{
  DataStream<Tuple2<String, Integer>> wordByWordStream = wordDataSourceStream.flatMap(new SplitSentenceTransformation()).name("WordByWordStream");
  return wordByWordStream;
}

// ...

package com.sampleFlinkProject.transformations;

import org.apache.flink.api.common.functions.FlatMapFunction;
import org.apache.flink.api.common.functions.RichFlatMapFunction;
import org.apache.flink.api.java.tuple.Tuple1;
import org.apache.flink.api.java.tuple.Tuple2;
import org.apache.flink.util.Collector;

public class SplitSentenceTransformation extends RichFlatMapFunction<String, Tuple2<String, Integer>> {

    @Override
    public void flatMap(String input, Collector<Tuple2<String, Integer>> collector) throws Exception
    {
        String[] splittedSentence = input.toLowerCase().split("\\W+");
        for (String eachWord : splittedSentence)
        {
            collector.collect(new Tuple2<String, Integer>(eachWord, 1));
        }
    }
}
```

I have created another method in the class `com.sampleFlinkProject.streams.StreamCreator.splitSentenceWordByWord()`

This method takes our data source stream and convert it to the another stream whose type **Tuple2<String, Integer>>.** I have chosen the tuple because to sum all the words(Please keep reading).

Function to be applied to the data source is `SplitSentenceTransformation` . Here I am going to split each sentence then put them in a collector. Values in the collector will be input for our new data stream `wordByWordStream` . Each input will be like: (cat, 1), (pat, 1), ({word}, 1) …

After that I going to filter(remove) the wordByWordStream. My filter function basically will do: “If word contains the specific character(return true), keep in the stream, otherwise remove it(return false)”

```java
public DataStream<Tuple2<String, Integer>> filterWordThatContainsSpecificCharacter(DataStream<Tuple2<String, Integer>> wordByWordStream, String specificCharacter)
{
  DataStream<Tuple2<String, Integer>> filteredStream = wordByWordStream.filter(new FilterStreamByCharacter(specificCharacter)).name("Filter Stream By Specific Character");
  return filteredStream;
}

// ...

public class FilterStreamByCharacter implements FilterFunction<Tuple2<String, Integer>>
{
    private Logger logger = LoggerFactory.getLogger(FilterStreamByCharacter.class);

    private String specificCharacter;

    public FilterStreamByCharacter(String specificCharacter)
    {
        this.specificCharacter = specificCharacter;
    }

    @Override
    public boolean filter(Tuple2<String, Integer> value) throws Exception
    {
        String wordInTuple = value.getField(0);

        return wordInTuple.contains(specificCharacter);
    }
}
```

I think code is self-explanatory.

Lastly, I am going to convert this stream to the KeyedStream and sum the words.

```java
public SingleOutputStreamOperator<Tuple2<String, Integer>> sumTheWordContainsSpecificCharacter( DataStream<Tuple2<String, Integer>> filteredStream)
{
  SingleOutputStreamOperator<Tuple2<String, Integer>> sumWordStream = filteredStream.keyBy(1).sum(1).name("Count keyed stream");
  return sumWordStream;
}
```

Now, let’s explain some important points:
- `filteredStream.keyBy(1)` : means that group the stream according to the first position of the tuple which is the Integer part. And remember that our input stream data pattern is like that: (word, 1), (word2, 1) etc.. with grouping them by first field, actually I am taking all the elements.
- `filteredStream.keyBy(1).sum(1)` : means that “sum” the keyed input stream according to the first position of the field which is the Integer part again. Therefore “sum” will be counting all the words in the stream. For example, if input values are (word, 2), (word2, 2), then our result will be double of the correct one.

<br />

## Step 4 — Writing the Result

In this series, we are just printing the result instead of writing to the Kafka, mongodb etc..

```java
summedStream.print();
```


Last but not least, wait for the next post …