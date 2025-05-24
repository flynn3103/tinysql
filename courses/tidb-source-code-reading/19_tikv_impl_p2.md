# TiDB Source Code Reading Series Article: TiKV Client Implementation (Part 2)

This article continues to introduce two main modules in tikv-client: the copIterator responsible for distributed computing and the twoPhaseCommitter for handling the second phase of transactions.

## copIterator

### Overview

Before introducing the concept of copIterator, we need to briefly review the concepts of distsql and coprocessor and their relationship with SQL statements from [TiDB source code reading series article (6)](https://cn.pingcap.com/blog/tidb-source-code-reading-6/).

tikv-server supports SQL layer computing capabilities through the coprocessor interface. Most commonly used calculations that only involve single-table data can be pushed down to tikv-server. After calculation, although the data read from the storage engine remains the same, the data returned through the network will be much less, greatly saving serialization and network transmission costs.

distsql is an abstraction layer between the SQL layer and coprocessor. It encapsulates the lower coprocessor request to provide a simple upper layer `Select` method for performing single table calculation tasks. While top-level SQL statements may contain complex `JOIN`, `SUBQUERY` matches involving many tables, distsql only involves data from a single table. A distsql request will involve multiple regions, and we will execute a coprocessor request for each region involved.

So their relationship is: one SQL statement contains multiple distsql requests, and one distsql request contains multiple coprocessor requests.

**The task of copIterator is to implement distsql requests, execute all involved coprocessor requests, and return results in sequence.**

### Constructing coprocessor tasks

A distsql request needs to process data from a single table's index scan or table scan, with the Request containing the converted KeyRange list. Next, through the [LocateKey](https://github.com/pingcap/tidb/blob/v2.1.0-rc.2/store/tikv/region_cache.go#L138) method provided by region cache, we can find which regions contain data within a key range.

After finding all regions containing all KeyRanges, we need to split the key range list according to region ranges, so that each coprocessor task's key range list won't exceed the region's range.

After constructing all coprocessor tasks, the next step is to execute these tasks.

### copIterator execution modes

To better understand copIterator's execution modes, let's start from the simplest implementation and gradually evolve to the current design.

copIterator implements the `kv.Response` interface, needing to implement the corresponding [Next](https://github.com/pingcap/tidb/blob/v2.1.0-rc.2/store/tikv/coprocessor.go#L516) method. When the upper layer calls Next, it returns a [coprocessor response](https://github.com/pingcap/tidb/blob/v2.1.0-rc.2/store/tikv/coprocessor.go#L390:6). The upper layer gets multiple coprocessor responses by calling `Next` multiple times until all results are retrieved.

The simplest implementation is to execute one coprocessor task in the `Next` method and return that task's execution result.

A major problem with this execution mode is that a lot of time is spent waiting for coprocessor request results. We need to improve this.

If coprocessor requests are triggered by `Next`, each `Next` call must wait for one `RPC round trip` delay. We can modify it so requests are triggered before `Next` is called, allowing results to be obtained earlier when Next is called, eliminating blocking wait time.

When creating copIterator, we start a background worker goroutine to execute all coprocessor tasks sequentially and send execution results to a response channel. This way the foreground `Next` method only needs to receive a coprocessor response from this channel. If a task has completed execution, the `Next` method can get the result immediately and return.

When all coprocessor tasks are completed by the worker, the worker closes the response channel. When `Next` receives from the closed channel, it returns `nil response` indicating all results are processed.

The above execution scheme still has an issue - coprocessor tasks are executed by only one worker without parallelism, so performance is not ideal.

To increase parallelism, we can create multiple workers to execute tasks, send all tasks to a task channel, have multiple workers read tasks from this channel, and after completion send results to the response channel. The number of workers controls concurrency.

After this modification, execution can be fully parallel, but it brings a new problem - tasks are ordered but responses are out of order due to parallel execution by multiple workers. This execution mode works for distsql requests that don't require ordered results. For distsql requests requiring ordered results, we need another execution mode.

Currently when a worker completes a task, the response is sent to a global channel. If we create a channel for each task and send responses to that task's own response channel, Next can get responses in task order by retrieving from channels in sequence, ensuring ordered results.

This is copIterator's final execution mode.

### copIterator implementation details

After understanding the execution modes, let's analyze the complete execution flow from the source code perspective.

#### Frontend execution flow

The first step of frontend execution is CopClient's [Send](https://github.com/pingcap/tidb/blob/v2.1.0-rc.2/store/tikv/coprocessor.go#L82) method. It first [constructs coprocessor tasks](https://github.com/pingcap/tidb/blob/v2.1.0-rc.2/store/tikv/coprocessor.go#L238) from the KeyRanges in the distsql request, [creates copIterator](https://github.com/pingcap/tidb/blob/v2.1.0-rc.2/store/tikv/coprocessor.go#L88) with the constructed tasks, then calls copIterator's [open](https://github.com/pingcap/tidb/blob/v2.1.0-rc.2/store/tikv/coprocessor.go#L438) method to start multiple background [worker goroutines](https://github.com/pingcap/tidb/blob/v2.1.0-rc.2/store/tikv/coprocessor.go#L452), then starts a [sender](https://github.com/pingcap/tidb/blob/v2.1.0-rc.2/store/tikv/coprocessor.go#L454) to put tasks into the task channel, finally returns copIterator as `kv.Response`.

The second step of frontend execution is calling `kv.Response`'s `Next` method multiple times until getting all responses.

copIterator will choose the corresponding execution mode in `Next` based on whether results need to be ordered - unordered requests [get results from the global channel](https://github.com/pingcap/tidb/blob/v2.1.0-rc.2/store/tikv/coprocessor.go#L526), ordered requests get results from [each task's response channel](https://github.com/pingcap/tidb/blob/v2.1.0-rc.2/store/tikv/coprocessor.go#L537).

#### Backend execution flow

After [getting a task from the task channel](https://github.com/pingcap/tidb/blob/v2.1.0-rc.2/store/tikv/coprocessor.go#L417), the worker will execute [handleTask](https://github.com/pingcap/tidb/blob/v2.1.0-rc.2/store/tikv/coprocessor.go#L424) to send RPC requests and handle request errors. When a region splits, we need to [construct new tasks](https://github.com/pingcap/tidb/blob/v2.1.0-rc.2/store/tikv/coprocessor.go#L572) and resend them. For ordered distsql requests, results from multiple split tasks need to be sent to the old task's response channel, so one task's response channel may return multiple responses. After sending completes, the task's response channel needs to be [closed](https://github.com/pingcap/tidb/blob/v2.1.0-rc.2/store/tikv/coprocessor.go#L428).

## twoPhaseCommitter

### 2PC Introduction

2PC is a way to implement distributed transactions, ensuring atomicity of transactions spanning multiple network nodes so transactions won't be partially committed.

In TiDB, the 2PC model used is the Google percolator model. Simply put, the main difference between the percolator model and traditional 2PC is eliminating the transaction manager single point by saving transaction state information on each key, greatly improving distributed transaction linear scale capability. Although there is still a timestamp oracle single point, because the logic is very simple and can be batch executed, it won't become a system bottleneck.

For details about the percolator model, refer to this article: [https://cn.pingcap.com/blog/percolator-and-txn/](https://cn.pingcap.com/blog/percolator-and-txn/)

### Constructing twoPhaseCommitter

When a transaction is ready to commit, a [twoPhaseCommiter](https://github.com/pingcap/tidb/blob/v2.1.0-rc.2/store/tikv/2pc.go#L62) is created to execute the distributed transaction.

During construction, the following tasks need to be done:

1. [Collect all keys and mutations from `memBuffer` and `lockedKeys`](https://github.com/pingcap/tidb/blob/v2.1.0-rc.2/store/tikv/2pc.go#L91)

   `memBuffer` contains ordered keys, we traverse `memBuffer` sequentially to collect keys that need modification in the transaction. Entry with value length 0 indicates `DELETE` operation, value length > 0 indicates `PUT` operation. The first key in `memBuffer` serves as the transaction's primary key. `lockKeys` contains keys that don't need modification but need read locks, which will also be written to TiKV as `LOCK` operations in mutations.

2. [Check if transaction size exceeds limit](https://github.com/pingcap/tidb/blob/v2.1.0-rc.2/store/tikv/2pc.go#L132)

   When collecting mutations, the total transaction size is calculated. If it exceeds the maximum transaction limit, an error is returned.

   Too large transactions may put too much pressure on the TiKV cluster, causing execution failure and cluster unavailability, so transaction size needs a hard limit.

3. [Calculate transaction TTL time](https://github.com/pingcap/tidb/blob/v2.1.0-rc.2/store/tikv/2pc.go#L164)

   If a transaction's keys are locked through `prewrite` but the transaction doesn't complete and tidb-server crashes, other tidb-servers in the cluster cannot read these keys. Without TTL this would deadlock. With TTL, read requests can execute cleanup after TTL timeout and then read the data.

   Calculating a transaction's timeout needs to consider normal transaction execution time. If too short, large transactions cannot complete normally. If too long, abnormal exits may cause certain keys to be inaccessible for long periods. So this algorithm is used: TTL is proportional to the square root of transaction size and controlled between a minimum and maximum value.

### execute

After twoPhaseCommiter is created, the next step is executing the [execute](https://github.com/pingcap/tidb/blob/v2.1.0-rc.2/store/tikv/2pc.go#L562) function.

In the `execute` function, [cleanupKeys](https://github.com/pingcap/tidb/blob/v2.1.0-rc.2/store/tikv/2pc.go#L572) needs to be executed in the `defer` function to clean up extra locks when the transaction fails to execute. Without this step, remaining locks would block read requests until TTL expires. The first step executes [prewriteKeys](https://github.com/pingcap/tidb/blob/v2.1.0-rc.2/store/tikv/2pc.go#L585). If successful, gets a `commitTS` from PD for executing `commit` operations. After getting `commitTS`, the following validations are needed:

- `commitTS` is greater than `startTS`
- schema hasn't expired  
- transaction execution time isn't too long
- If checks don't pass, transaction fails with error

After passing checks, execute final step [commitKeys](https://github.com/pingcap/tidb/blob/v2.1.0-rc.2/store/tikv/2pc.go#L620). If no errors, transaction commits successfully.

When `commitKeys` requests encounter network timeout, whether the transaction has committed is uncertain. In this case `cleanupKeys` cannot be executed or it would break transaction consistency. We return a special [undetermined error](https://github.com/pingcap/tidb/blob/v2.1.0-rc.2/store/tikv/2pc.go#L625) for upper layer handling. The upper layer will disconnect on this error rather than return execution failure to the user.

[prewriteKeys](https://github.com/pingcap/tidb/blob/v2.1.0-rc.2/store/tikv/2pc.go#L533), [commitKeys](https://github.com/pingcap/tidb/blob/v2.1.0-rc.2/store/tikv/2pc.go#L537) and [cleanupKeys](https://github.com/pingcap/tidb/blob/v2.1.0-rc.2/store/tikv/2pc.go#L541) share much logic - needing to split keys into batches by region then execute one RPC per batch.

When RPC returns region expired error, we need to resplit that region's keys into batches and send RPC requests.

This logic is extracted into [doActionOnKeys](https://github.com/pingcap/tidb/blob/v2.1.0-rc.2/store/tikv/2pc.go#L191) and [doActionOnBatches](https://github.com/pingcap/tidb/blob/v2.1.0-rc.2/store/tikv/2pc.go#L239), implementing [prewriteSinlgeBatch](https://github.com/pingcap/tidb/blob/v2.1.0-rc.2/store/tikv/2pc.go#L319), [commitSingleBatch](https://github.com/pingcap/tidb/blob/v2.1.0-rc.2/store/tikv/2pc.go#L421), [cleanupSingleBatch](https://github.com/pingcap/tidb/blob/v2.1.0-rc.2/store/tikv/2pc.go#L497) functions for executing single batch RPC requests.

Although most logic is the same, different requests have different execution order requirements that need special handling in `doActionOnKeys`:

- Multiple `prewrite` batches need synchronous parallel execution
- For `commit` batches, first batch must execute successfully before asynchronously executing other batches in parallel  
- Multiple `cleanup` batches can execute asynchronously in parallel

[doActionOnBatches](https://github.com/pingcap/tidb/blob/v2.1.0-rc.2/store/tikv/2pc.go#L239:29) starts multiple goroutines to execute batches in parallel. If encountering error, it cancels other executing contexts and returns the first error encountered.

When executing `prewriteSingleBatch`, region split errors may occur where batch keys are no longer in one region. Here we recursively call [prewriteKeys](https://github.com/pingcap/tidb/blob/v2.1.0-rc.2/store/tikv/2pc.go#L352) to redo the batch splitting, `doActionOnBatch` and `prewriteSingleBatch` flow. This logic exists in `commitSingleBatch` and `cleanupSingleBatch` as well.

twoPhaseCommitter contains only a small part of the transaction model logic, with main logic in tikv-server, which is beyond this article's scope and won't be discussed in detail here.

> Click to see more [TiDB source code reading series articles](https://cn.pingcap.com/blog/tag/tidb-source-code-reading/) 
