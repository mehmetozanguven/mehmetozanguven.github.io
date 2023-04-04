---
layout: post
title: "Utility script for pretty printing kafka consumer log"
date: 2022-01-29 10:45:31 +0530
categories: "others"
author: "mehmetozanguven"
newUrl: "https://mehmetozanguven.com/pretty-printing-kafka-log/"
---

Recently, I had to listen kafka consumer log to identity bug I have found in my web application. But listening kafka consumer logs via console output was a pain and reading logs on the console was so hard. Then I was just starting to search "how to pretty print for the kafka logs". After all search, finally I created a util python script to listen the logs.

I also didn't want to see all fields inside the kafka record. I only wanted to see specific fields I was looking for.

If you faced the same problem, you can use this script as well

Script itself is self-explanatory, I will not explain it. You can run this script from your IDE (such as PyCharm).

```python
from subprocess import Popen, PIPE
import json
import datetime



def listen_console(command):
    process = Popen(command, stdout=PIPE, shell=True)
    while True:
        line = process.stdout.readline().rstrip()
        if not line:
            break
        yield line


def pretty_print(kafka_json):
    creation_date = datetime.datetime.fromtimestamp(float(kafka_json["{timestamp_field}"]) / 1000)
    print("{")
    print("Any field: ", kafka_json["field_name"])
    # ...
    print("}")


if __name__ == "__main__":
    kafka_consumer_log = "sh /path/to/kafka/bin/kafka-console-consumer.sh --topic {kafka_topic_name} --bootstrap-server {ip_address}:9092"
    for item in listen_console(kafka_consumer_log):
        kafka_json = json.loads(item)
        pretty_print(kafka_json)
```
