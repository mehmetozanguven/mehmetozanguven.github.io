---
layout: post
title:  "Apache Flink Series 1 — What is Apache Flink"
date:   2020-02-15 19:45:31 +0530
categories: "apache-flink"
author: "mehmetozanguven"
---

In this post, I will try to explain what is Apache Flink, what is used for, and features of Apache Flink.
<br />
<br />

> You may see the all my notes about Apache Flink with this [link](/apache-flink/)

## What is Apache Flink

- Apache Flink is a distributed stream processor to implement stateful or stateless processing applications
- Or in other definition could be: Apache Flink is a distributed processing engine for big data that performs stateful or stateless computations over both bound and unbound data streams. Apache Flink is deployed in various cluster environments for fast computations over data of different sizes.
- Simply put: With Apache flink, you have cluster which includes master and slaves machines and you give your job to master machine then slaves will do your job in efficient and fault-tolerant manner.

Before pass the “use cases for Apache Flink”, let me point to the **what does the stateful stream application means?**

### What is Stateful Stream Processing?

- Stateful stream processing is an application design pattern for processing unbounded streams of events.
- It means that data is created as continuous streams of events
- You can think of user interactions on website or in mobile apps, placements of orders, server logs or sensor measurements. All of these are stream of events.

### How Stateful stream processing works?

- When an application receives an event, it can perform arbitrary computations that involve reading data from or writing data to the state.
- State can be stored and accessed in many different places including program variables, local files or embedded or external databases

Now this leads us to another question :) What is a state?

### What is a state?

- At a high level, we can consider state as memory in operators in Flink that remembers information about past input and can be used to influence the processing of future input.
- State is data information generated during computations that plays a very important role in fault tolerance, failure recovery and checkpoints in Apache Flink

You may think what is a operator in Flink, I will explain it in another post. Right now, I can say that operator is related to the stream.

## Use cases for Apache Flink

- Real-time recommendations (recommending products while customers browse a retailer’s website)
- Pattern detection or complex event processing (fraud detection in credit card transaction)
- Anomaly detection (to detect attemps to intrude — accessing the restricted places — )
- Alternative to traditional approach to sync. data in different storage system. In traditional way there is periodic jobs called ETL(extract,transform,load). However they do not meet the latency requirements for many use cases.
- Flink also can be used for data analytics applications.


## Features of Apache Flink

- Exactly one state consistency guarantees
- Millisecond latencies while processing millions of events per second
- Easy to use
- Connectors to the most commonly used storage system such as Kafka, Cassandra, ElasticSearch
- Ability to run with very little downtime
- It also can be used for batch processor.
- Provides Event-time and processing-time semantics
    - Event-time semantics provide consistent and accurate results despite out of order events.
    - Processing-time semantics (Processing time refers to the system time of the machine that is executing the respective operation) can be used for applications with very low latency