---
layout: post
title:  "Apache Flink Series 9 - How Flink & Standalone Cluster Setup Work?"
date:   2020-05-04 01:30:31 +0530
categories: "apache-flink"
author: "mehmetozanguven"
---

In this post, I am going to explain, how Flink starts itself, and what happens when you submit your job to the Standalone Cluster setup

## Standalone Cluster

- Consists of at least one master process and at least one TaskManager process that run on one or more machines.
- All processes run as regular Java JVM process.
- The master process runs a Dispatcher and a ResourceManager in separate threads.
- Once they start running, the TaskManagers register themselves at the ResourceManager

<img src="/assets/apache_flink/standaloneSetup.png" alt="standaloneSetup.png" title="Standalone Setup" />

<br />

<br />

## What happens when you submit your job?

- A client submits a job to the Dispatcher, which internally starts a JobManager thread and provides the JobGraph for execution.
- The JobManager requests the necessary processing slots from the ResourceManager.
- ResourceManager requests slot(s) from TaskManagers
- TaskManager offers their free slots to the JobManager
- JobManager executes the tasks.
- In a standalone deployment, the master and workers are not automatically restarted in the case of failure.
- A job can recover from a worker failure if a sufficient #processing slots is available. This can be ensured by running one or more standby workers

<img src="/assets/apache_flink/submitJobStandalone.png" alt="submitJobStandalone.png" title="Submitting Job To Standalone Cluster" />

<br />

Last but not least wait for the next post...