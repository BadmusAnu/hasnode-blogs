---
title: "FluentBit Quick Dive"
datePublished: Sun Dec 21 2025 05:25:43 GMT+0000 (Coordinated Universal Time)
cuid: cmjfaaak0000302l78lyr9d1e
slug: fluentbit-quick-dive
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1766296179590/d7798381-2de3-4f1d-9410-ae28109fd9d8.png
tags: kubernetes, observability, fluentbit

---

Log Shippers are applications that serve one main purpose: to move log entries from an ephemeral (easily overwritten) location to a stage where they can be further processed (log aggregators e.g., Logstash) or permanently stored and visualized (log viewers e.g., OpenSearch). FluentBit is one of the most popular and mature open-source log shipping solutions, due to some simple reasons: lightweight, extensibility, and integration with other cloud-native tools.

### Setting Up FluentBit for Log Shipping

Log shippers, like all applications, benefit tremendously from proper optimization. Designing for all situations in your system helps get the best value from your setup and ensures your application is more observable. Situations like what happens when the aggregator or output is down, what protocol should I use when sending logs, what are the retry intervals, how will my log shipper properly parse logs for different applications and frameworks (Go, Java, Python, C++, etc.). For this, we will deep dive into the FluentBit config file, and we will look at some sections and parts of the config file.

### Serrvice, Log Parsing and Storage Configuration

Service section of the fluentbit config file, manages and defines global properties of the fluentbit application, configs such as daemon, log\_level (of the fluentbit application), parser file location, etc. can all be controlled from this section.  
  
Notably, this section also includes configuration for handling backpressure and enabling buffering. The storage-related settings help to manage and define a filesystem to be used by the service, which is used for buffering and backpressure handling. `storage.path` is configured here; it maps to a disk location that can be used as buffer storage in the situation where the downstream output is unavailable. By default, these logs are stored in RAM and then sent. This is ideal if the logs are there for a few seconds and don’t build up. When the downstream is unavailable, that changes; logs need a less expensive/volatile location to be held to prevent a blocked pipeline.

```json
[SERVICE]
    Flush               1
    Daemon              Off
    Log_Level           info
    storage.path        /var/log/flb-storage/
    storage.sync        normal
    storage.checksum    off

    Parsers_File        parsers.conf
```

Log parsing is the syntactic analysis of log inputs to put them into organized chunks that allow them to be easily searched, sorted, and understood.

It’s possible to say a log is mainly broken down into four main sections: Time & Date - this defines the period the event occurred, log\_level - usually used as the severity level of a log which often dictates how important that event is, the message of the log itself sometimes includes a stack trace, and other metadata that define other information deemed necessary in analyzing the logs effectively.

A parser needs to be able to analyze, understand, and assign these labels to sections or parts of a log. Here are some sample logs and a sectionalization of each part; some are multiline, and the parser should be able to determine the start and stop location of a log entry. Below are examples of log entries to try to familiarize the reader with what the parser needs to do, the parser employed should accurately be able identify if a log is a single line log, like the json log below. The java log example also show the common method in which java application logs are output, where the log\_level, main error and stack strace are part of a multiline logging event. Python application often use indented space to show Traceback, and the pipe operator (|) to sperate sections like date and log\_level. Parsers must understand all these.

```python
2025-05-15 10:31:12,452 | CRITICAL | The parser is now locked onto this timestamp format.
Traceback (most recent call last):
  File "worker.py", line 42, in process_image
    # Because I am indented with spaces, the multiline filter knows I am not a new log.
    # I am part of the 'Traceback' payload belonging to the timestamp above.
    img.allocate(512MB)
MemoryError: The parser will only 'flush' this entire block to Logstash once it stops seeing indented lines or hits a timeout.
```

```java
[2025-05-15 14:00:01] ERROR [InventoryService] - The parser sees the bracketed timestamp above as the 'Start State' of a new record.
java.lang.NullPointerException: This line does not start with a bracket, so the parser assumes I am a continuation of the previous line.
    at com.shop.Inventory.update(Inventory.java:42)
    at com.shop.InventoryController.postUpdate(InventoryController.java:15)
Caused by: com.shop.exceptions.ValidationException: I will keep being 'swallowed' into the first log entry until the parser sees a brand new line starting with a '[' character.
```

```json
{"ts": "2025-05-15T14:10:05Z", "lvl": "INFO", "msg": "I am a single-line JSON object.", "parsing_note": "Fluent Bit does not need multiline logic for me because my entire stack trace is escaped into a single string field.", "trace": "java.lang.Exception: Stack trace content\n  at main.Execution(app.java:10)\n  at internal.Call(app.java:5)"}
```

Sometimes, if your application logs are not in a standard format, there might be a need to create a custom parser that understands the log structure. But more often than not, default parsers are very effective when properly defined. They can be defined in both the INPUT section and sometimes at the FILTER section if multiline parsers are employed. When multiple parsers are specified, log events are evaluated across each parser until a match is found. Attached below is a great guide for building custom parsers.  

%[https://youtu.be/XYh9wb29XPQ] 

### Parsing Log Entries with Multiline and Single-Line Formats

This defines the entry of logs into FluentBit, what kind of logs are extracted. Different input types are specialized for different kinds of log retrieval and use appropriate techniques to retrieve log data. Some inputs include tail, which simply keeps tailing a file continuously, stdin, which monitors the stdin path on your OS, and Kafka, which monitors any specified Kafka instance/url and can subscribe to specific topics and ingest from there. There are many input types and a full list can be retrieved from the [official docs](https://docs.fluentbit.io/manual/data-pipeline/inputs) of fluentbit  
Multiple inputs can be defined in a single config file, and this is especially helpful if you are trying to attach different tags to each log event from when they are ingested.

Log events can have specific tags assigned to them by the input section by specifying “Tag,” which can be used as a grouping basis to further apply specific operations to specific groups. The code below shows the use of the `kube.*` tag to mark all logs from the input section.

```plaintext
[INPUT]
    Name                tail
    Path                /var/log/containers/*.log
    Tag                 kube.*
    storage.type        filesystem
    DB                  /var/log/flb_kube.db
    Mem_Buf_Limit       50MB
    Skip_Long_Lines     On
```

### Filtering Logs with Tags

FluentBit provides a way to either enrich, declutter, drop, or further alter log event/data using the filter section. It defines single or multiple extra sets to be performed on log events before they are finally sent to the output. Common filters include modify, Kubernetes, grep, rewrite\_tag, Throttle, etc. Each of these filters can be applied to log events, and multiple filters can be used and chained in a single config. A full list of all filters can be retrieved fom the official documentation of fluentbit as linked aboved.

The Lua filter provides a method to introduce your own custom code to alter the log event. A Lua script is written, and the location is provided to work on all log events that match the specified tag or all logs if so specified.

```plaintext
[FILTER]
    Name                multiline
    Match               kube.*
    multiline.key_content log
    multiline.parser    java, python, go

[FILTER]
    Name                kubernetes
    Match               kube.*
    # Merges the 'log' JSON field if the app outputs JSON
    Merge_Log           On
    Keep_Log            Off
    K8S-Logging.Parser  On
    K8S-Logging.Exclude On

[FILTER]
    Name                modify
    Match               kube.*
    Remove              container_id
    Remove              pod_id
    # Remove            kubernetes.labels.some-noisy-label
```

### Configuring Output Settings

This defines your downstream or next stage of your log ingestion. Some popular outputs are sysout, http, Elasticsearch, Prometheus, etc. You can send logs to multiple outputs as well as configure output settings like TLS, formats, ports, routing, etc.  
Routing is done with the Match key in the output, and it’s based on the Tags, but also routing based on regex match of the tags is possible.

```plaintext
[OUTPUT]
    Name                http
    Match               kube.*
    Host                logstash.your-namespace.svc
    Port                8080
    Format              json
    Retry_Limit         False
    storage.total_limit_size 5G
```

### Setting Up FluentBit

FluentBit can be installed in a number of ways, in a Kubernetes cluster using Helm, Docker, or Linux source build and install. The official Docs contain a guide on all methods of installation. For the benefit of this tutorial, we will be using Helm and installing the AWS-maintained version of FluentBit that contains some built-in extra plugins.

`helm repo add eks` `https://aws.github.io/eks-charts`

`helm repo update`

`helm upgrade --install aws-for-fluent-bit eks/aws-for-fluent-bit --namespace kube-system --values values.yaml`

```yaml
# values.yaml for eks/aws-for-fluent-bit version 0.1.35
# Ref: https://artifacthub.io/packages/helm/aws/aws-for-fluent-bit?modal=values

# --- Global Service Settings ---
service:
  # Using extraService for custom storage and parser definitions
  extraService: |
    Flush               1
    Daemon              Off
    Log_Level           info
    storage.path        /var/log/flb-storage/
    storage.sync        normal
    storage.checksum    off
    Parsers_File        parsers.conf

# --- Optimized Input Settings ---
# We leverage the chart's built-in 'tail' configuration
input:
  path: "/var/log/containers/*.log"
  tag: "kube.*"
  db: "/var/log/flb_kube.db"
  memBufLimit: "50MB"
  skipLongLines: "On"
  # Enabling filesystem storage type for the tail input
  extraParams: |
    storage.type        filesystem

# --- Filter Chain ---
filter:
  # Using extraFilters to define the multiline, kubernetes, and modify chain
  extraFilters: |
    [FILTER]
        Name                multiline
        Match               kube.*
        multiline.key_content log
        multiline.parser    java, python, go

    [FILTER]
        Name                kubernetes
        Match               kube.*
        Merge_Log           On
        Keep_Log            Off
        K8S-Logging.Parser  On
        K8S-Logging.Exclude On

    [FILTER]
        Name                modify
        Match               kube.*
        Remove              container_id
        Remove              pod_id

# --- Output Settings ---
# Disable default AWS outputs to route exclusively to Logstash HTTP
cloudWatch:
  enabled: false
firehose:
  enabled: false
kinesis:
  enabled: false
elasticsearch:
  enabled: false

output:
  extraOutputs: |
    [OUTPUT]
        Name                http
        Match               kube.*
        Host                logstash.your-namespace.svc
        Port                8080
        URI                 /fluent-bit
        Format              json
        Retry_Limit         False
        storage.total_limit_size 5G

# --- Persistence and RBAC ---
extraVolumes:
  - name: flb-storage
    hostPath:
      path: /var/log/flb-storage
      type: DirectoryOrCreate

extraVolumeMounts:
  - name: flb-storage
    mountPath: /var/log/flb-storage

rbac:
  create: true

serviceAccount:
  create: true
```