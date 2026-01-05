---
title: "AWS OpenSearch Quick Dive"
datePublished: Sun Jan 04 2026 21:33:21 GMT+0000 (Coordinated Universal Time)
cuid: cmk090lxl000c02jx8bfw74qq
slug: aws-opensearch-quick-dive
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1767648198437/bdc8f4a9-742c-4e38-b70e-fa3a34295fa7.png
tags: aws, programming, logging, opensearchservice

---

Analyzing logs is one of the most common tasks developers perform regularly, which also takes on a special meaning during application outages and downtime. This article explores using AWS OpenSearch Service (closely related to the open-source Elasticsearch) to ingest logs. We will explore various approaches to log lifecycle management, plan your OpenSearch indexing and sharding, some simple tips to fix issues, and access logs from a developers pov, let’s get into it.

Problem Statement  
I will be describing a situation (common enough in medium-sized engineering companies), we will design and plan our OpenSearch setup to address this, will also discuss some general OpenSearch terms that can guide our design

A medium-sized company with 5 engineering teams, with each team owning one or more services, needs all developers be able to access logs from a centralized location, and each team must keep logs for a minimum of 120 days. Teams can choose to create tenants (silos containing just logs relating to their team), and a global view where all logs can potentially be found. Developers should be able to easily filter logs that concern them, the log searching rate should be quick and allow multiple users, and indexing (log addition) should be easy enough. Averagely each service (application) produces 2GB worth of logs per day.

## Cluster Design and Indexing

AWS OpenSearch is mainly a fixed cluster, i.e., the number of nodes doesn’t grow or shrink with load. In our design, we will be working with 5 node cluster, each with 100GB volumes attached to it (giving a total of 500 GB), primaries and replica counts are 5 and 1, respectively. To explain this, we discuss some OpenSearch terminology briefly

### Terminology:

* Documents: A document is a basic unit that stores information (text or structured data), think of it as a single page in a book
    
* Shard: This is a standalone, fully functional search engine, think of a complete chapter in a book.
    
* Index: This is a logical grouping of data, a collection of documents, that defines schemas (mappings), settings (replica and primary counts), etc. Think of the entire book, including the appendix and content pages.
    
* Primary Count: This is the number of shards your index will be split into and distributed across the cluster. 5 primaries mean a single index will be split into 5 shards distributed across the cluster. Think of the number of "buckets" I’m using to divide my total data, so multiple servers can help store and search it.
    
* Replica Count: This is exact number of copies of the shard that will be kept. 1 replica means a single extra copy of your shard will be kept on the cluster. think of the number of backup buckets I have for traffic and data security reasons
    

Primary and Replica Counts affect how quickly data is indexed (stored and added to the cluster ) and how quickly queries are handled and search results returned. The simplest description is **Primaries** let you store more data and write it faster by dividing it; **Replicas** let you serve more users and stay online by duplicating it.

Indexing design; we intend to ensure each service gets its own index, i.e., index 1 - “team-red-service-a”, index 2- “team-red-service-b”, index 3 team-blue-service-c”, etc. This design helps with possible issues like mapping conflicts, search flexibility, service-based lifecycle management, etc.  
For OpenSearch optimal performance, we are encouraged to observe a couple of constraints.

*A single shard is within the 10 to 50GB ratio (not more than 200 million documents per shard)\*.*

*A single node has a limit of the number of shards hard limit of 1000 shards on older versions and 4000 on newer versions, in-fact a soft limit with the 25-shard-rule is strongly suggested, i.e., for every 1GB of JVM Heap Memory, not more than 25 shards.\*\**

Consequently, these constraints both lead to the fact that indices need to be lifecycled, i.e., a single index should go through the following stages: opened (created and write allowed), search-only (write disallowed, but still queryable), closed (not queryable, but still exists, and can be reopened), and deleted. These stages ensure we have data constantly moving, freeing up disk space, compute costs like memory, and improving our cluster query response time. For these, we need to understand lifecycle management in OpenSearch

## Lifecycle Management

When dealing with Time-Series data, like our logs, we will opt for an OpenSearch feature called Data Stream, which provides us with a good abstraction on managing indices especially rollovers.  
DataStream is an abstraction layer for managing append-only time-series data across multiple "backing indices." One of the core caveats of datastream is that a date-time field must always be present, and once defined, that field can’t change. That is the reason why there only usable with time-series data.

A complete workflow of using datastreams, your logs when first received will be stored on a hidden index called .ds-team-red-service-a-000001. When the size of this index reaches 30GB, the logs are rolled over, and a new index called .ds-team-red-service-a-000002 is created. The initial index -000001 remains queryable, but data can’t be appended to it anymore, hence it is now a readonly state, this matters because after the index is moved to readonly state, it’s common to change values like force merge segment, to ensure data compatibility or number of replica to zero, has it’s expected that those logs are less important and the risk are worth the disk and compute saves.  
Each index contains certain metadata that determines if it’s open, closed, writable, etc. The tag that determines which of all these indices is currently active for write is the `"is_write_index": true`When data needs to be written, OpenSearch looks through the backing indices (.ds-team-red-service-a-000001, .ds-team-red-service-a-000002, etc) and all we have `"is_write_index": false` except the currently active index.

### Some important terminologies in Lifecycle Management

Index Templates: Predefined blueprints used to automatically initialize new indices with consistent settings (shards, replicas) and mappings. It’s essentially the glue of this process; the datastream is enabled here, and the corresponding ISM template to attach is also specified here, as shown in the code examples below.

ISM Policy: An automation engine that uses "states," "actions," and "transitions" to manage an index throughout its life (e.g., Hot → Warm → Delete).

```json
PUT _plugins/_ism/policies/team_log_policy
{
  "policy": {
    "description": "120-day lifecycle: Hot -> Warm (Search Only) -> Delete",
    "default_state": "hot",
    "states": [
      {
        "name": "hot",
        "actions": [
          {
            "rollover": {
              "min_index_age": "1d",
              "min_primary_shard_size": "30gb"
            }
          }
        ],
        "transitions": [
          {
            "state_name": "warm"
          }
        ]
      },
      {
        "name": "warm",
        "actions": [
          { "read_only": {} },
          { "force_merge": { "max_num_segments": 1 } }
        ],
        "transitions": [
          {
            "state_name": "delete",
            "conditions": {
              "min_index_age": "120d"
            }
          }
        ]
      },
      {
        "name": "delete",
        "actions": [
          { "delete": {} }
        ],
        "transitions": []
      }
    ],
    "ism_template": {
      "index_patterns": ["team-*"],
      "priority": 100
    }
  }
}
```

```json
PUT _index_template/team_logs_template
{
  "index_patterns": ["team-*"],
  "data_stream": { },
  "priority": 100,
  "template": {
    "settings": {
      "index.number_of_shards": 5,
      "index.number_of_replicas": 1,
      "plugins.index_state_management.policy_id": "team_log_policy"
    },
    "mappings": {
      "properties": {
        "@timestamp": { "type": "date" },
        "service_name": { "type": "keyword" },
        "team": { "type": "keyword" },
        "message": { "type": "text" }
      }
    }
  }
}
```

The image below shows a full relationship of the lifecycle, as well as the processes that take place at certain stages.

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1767561280046/507eee94-0f8c-47cc-9ee1-12bc503df1c7.png align="left")

## Log Retrieval: Developer Experience (DX)

Our problem statement requires making log retrieval as straightforward as possible for five different engineering teams. To achieve this, we leverage **Tenants**, **Index Patterns**, and **Dashboards**. Below this section, we have some UI images about the different tools we are discussing.

The **Global Tenant** is the default, shared workspace in OpenSearch. In our design, the Global Tenant satisfies the requirement for a "global view where all logs can potentially be found." It is the one place where an SRE or Architect can build a single dashboard that correlates logs across all five teams to debug cross-service outages.

**The "Team Tenant" (Silos):** Conversely, each team should have their own private tenant. This acts as a personalized silo where they only see their own index patterns and dashboards, preventing the UI from becoming cluttered with other teams' data.

An **Index Pattern** is a regex-based filter that allows you to query multiple backing indices at once.

> NB: when new fields are added to the logs, this fields are not automatically searchable or visible on the index patterns, and it’s necessary to refresh the index pattern so this can be searchable.

**Matching Data Streams:** Since our Data Stream creates hidden indices like `.ds-team-red-service-a-000001`, a developer simply creates a pattern like `team-red-service-a-*`. This pattern "follows" the data as it rolls over from Hot to Warm.

**Team-Wide Patterns:** A developer can also create a broader pattern `team-red-*` to see all logs from every service their team owns in one single view.

**Dashboard:** An OpenSearch Dashboard is a collection of visualizations, searches, and maps organized into a single pane. Dashboards provide at-a-glance insights into your data and allow users to filter through millions of logs using a point-and-click interface. It can be used by the team to aggregate and provide a single point filter for log fields like `“log_level” : ERROR`

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1767644965278/567ff2ec-f00b-4aa5-8799-b20d280cf9e8.png align="center")

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1767644975141/cf51ac95-5788-4b28-b3da-9fda5f81f180.png align="center")

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1767645221712/900a7999-5658-46b3-84b1-f5783b513878.png align="center")

In our discussion, we have important concepts like cluster design, lifecycle management, sharding indexing, and log retrieval for developers, with concepts we have been able to design and provision a stable and highly available cluster.  
Some further topics that can be explored outside of this article about OpenSearch are RBAC, OpenSearch API, and how they can be used for data reindexing, cloning, and other tasks, etc.

Included below are some official documentation and guides that can provide additional information.

Thank you for reading this version of Quick Dive. See you at the next one.

[AWS OpenSearch Sharding](https://docs.aws.amazon.com/opensearch-service/latest/developerguide/bp-sharding.html), [Official OpenSearch Documentation](https://docs.opensearch.org/)