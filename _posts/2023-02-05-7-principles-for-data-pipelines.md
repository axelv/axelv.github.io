---
layout: post
title: "7 principles to build effective data pipelines"
permalink: /posts/prinicples-for-data-scientists
---

# Principles for data dipelines

Data pipelines are hard. Like every piece of software they fail, especially when the pipeline is in early development or the environment is changing rapidly. And on top of that development iterations are slow because of the execution time the pipeline takes. To speed up things most data pipelines make use of some form of multiprocessing which makes them even harder debug. To tackle these issues we want the be able to reëxecute parts of a data pipeline. This means (1) splitting the pipeline in pieces with a clear signature that enables us to re-run them seperatly and (2) persisting intermediate results so we can re-execute that part of the pipeline where things went wrong.

## Seperation in idempotent stateless tasks with intermediate persistence

The obvious thing to do is splitting the pipeline in pieces using functions. It's the first level of abstraction to reach for when trying seperate concerns. It makes pieces of your data processing reuseable and more readable.
But data pipelines as written as functions have one major problem: they don't differentiate between parameters and data input. In order to make the signature more explicit we'll use _dataclasses_. This allows us to seperate the parameters from the data input:

```python
    from dataclasses import dataclass

    @dataclass
    class MyDataProcesssingTask:
        a: int
        b: str

        def __call__(self, data):
            # the magic performing your data transformation

```

The pipeline should overwrite data if it is reexecuted and stale data exists and shouldn't have side effects. If a pipeline creates a new record in a SQL-database then it should use `UPSERT` instead of `INSERT`. Files should be overwritten be reusing

    In order to create a clear signature and seperate the parameters from data input we'll use Python's dataclass:

    ```
    class MyTask()


    ```

## Idempotency requires data partitions with deterministic keys

    key = function of the task and its parameters.
    Solution is dataclasses => hashable, repr, automatic __init__

## Atomicity, completeness of a task and re-execution

    In previous paragraph we discussed idempotentcy...
    Now we will discuss how this affects `completeness` of a task and wether we have to executed again or if output data is available and valid.

    One important aspect to enable correct evaluation of completeness is atomicity of write operations. The evaluation rule of completeness should only become true when the data is written available for read operations. This sounds obvious but a lot of datastores use some form of cache or create a data partition before bytes are written.
    Ex. Files. SQL Transactions, or Elasticsearch commit log and flush

## Serialization of tasks

    Serialization is importent for reliable scheduling and multiprocessing. But it also opens the door to put tasks on message brokers to use different backends for distributed computing.

## Scheduling a DAG of tasks for multiple workers

    Queueing tasks given its directed-acyclic graph of task dependecies

## Static dependencies, runtime dependecies, dynamic depencies

## Optimizing execution on workers by batching tasks

## Cleanup stale artifacts

When syncing partitions between two systems, ex. you transform a files in a source directory and save each of them in a new target directory, it's hard keep removed files in sync. You end up with stale transformed files in the target directory. In order to determine wether the target file should still exist, you have to iterate over all of them and check if the corresponding source file still exists. This is countercurrent to our typical data transform flow and involves a lot code and work for a simple cleanup operation. A solution is to work with a deletion-less system where the data partitions (in this example files) are not removed but marked as _stale_ and propagate the stale status.

If necessary a clean-up job can the remove all stale data partitions, under the assumption the stale status is in sync.

[comment]: <> (May be we should think differently about the data paritions and see them more as data updates operations. Then the _stale_ flag becomes a deletion operation. The whole pipeline is more about propagating/syncing operation.)

## Cleanup intermediate artifacts

    The maybe undesired side effect of intermediate persistance is

# a sequence of data operations

- Given an list of posts that have likes
- for each user, count the likes
- for each user, average the top 3 most liked posts
- find the top 10 users with the highest average

#
