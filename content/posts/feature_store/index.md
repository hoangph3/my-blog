---
title: "Building a Feature Store for Machine Learning"
date: 2023-07-29T13:32:54+07:00
tags: ["MLOps", "Machine Learning"]
author: ["hoangph3"]
showToc: true
TocOpen: false
draft: false
hidemeta: false
comments: false
description: "Design and proposal for a feature store to serve consistent features for training and serving."
summary: "Design and proposal for a feature store to serve consistent features for training and serving."
disableHLJS: false
disableShare: false
hideSummary: false
searchHidden: false
ShowReadingTime: true
ShowBreadCrumbs: true
ShowPostNavLinks: true
ShowWordCount: true
ShowRssButtonInSectionTermList: true
UseHugoToc: true
---

## Feature Store with Spark Streaming, Kafka, DataLake

Feature Store provides a centralized location to store and document features that will be used in machine learning models and can be shared across projects.

A Feature Store solution might address one or a combination of these problems:

- Feature management:
A feature store can help teams share and discover features, as well as manage roles and sharing settings for each feature (feature catalog).
- Feature computation:
A feature store can help with both performing feature computation and storing the results of this computation (data warehouse).
- Feature consistency:
A key selling point of modern feature stores is that they unify the logic for both batch features and streaming features, ensuring the consistency between features during training and features during inference.

![Source: https://www.tecton.ai/blog/what-is-a-feature-store/](feature-store.svg)

## Proposal

## Feature Store proposed architecture

![](proposed-architecture.png)

The solution currently implemented contains only the following components:

- Producer: applications producing entities to the kafka.
- Registry: repository of the schemas created on the pipeline stages.
- Kafka: streaming platform used to enable spark jobs to transform the entities produced by applications into features used by ML models. a high-performance data pipeline.
- Transformations: spark jobs to transform the entities produced by applications into features used by ML models. (User defined functions).
- Sinks: application that syncs the offline feature to the online store.
- Redis: in-memory data structure store, used as an online storage layer (caching).

![](cache.png)

1. Deploy:

```sh
make start
```

1. Produce logs to kafka:

```sh
make produce
```

1. Sink jobs:

```sh
make sink
```

1. Access the UI:

- [Kafdrop ui](http://localhost:9000)
- [Transformations spark ui](http://localhost:4040/StreamingQuery)
- [Mongodb ui](http://localhost:8081)
- [Redis ui](http://localhost:8082)

1. Clean:

```sh
make clean
```

You can get full source code here: [feature-store](https://github.com/hoangph3/mlops-labs/tree/main/feature-store), [proposal](https://github.com/hoangph3/mlops-labs/tree/main/feature-store/proposal).
