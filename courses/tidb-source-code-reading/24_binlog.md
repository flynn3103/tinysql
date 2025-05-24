# TiDB Source Code Reading Series Article: TiDB Binlog Implementation

This article introduces how TiDB sends binlog data to the Pump component of the TiDB Binlog cluster during the execution of DML/DDL statements.

## TiDB Binlog Overview

This article is not about the source code of the TiDB Binlog component, but about how TiDB sends binlog data to the Pump component of the TiDB Binlog cluster during the execution of DML/DDL statements. Currently, TiDB's binlog for DML is similar to MySQL's [Row-based](https://dev.mysql.com/doc/refman/5.7/en/binary-log-formats.html) format. For specific TiDB Binlog architecture details, please refer to [TiDB Ecosystem Tools Original Understanding Series (1): Principles of Evolution and Realization of TiDB-Binlog Architecture](https://cn.pingcap.com/blog/tidb-ecosystem-tools-1/).

**Here we only describe the implementation of the code in TiDB.**

## DML Binlog

TiDB uses protobuf to encode binlog, the specific format can be seen in [binlog.proto](https://github.com/pingcap/tipb/blob/master/proto/binlog/binlog.proto). Here we discuss the mechanism by which TiDB writes binlog and the impact of binlog on TiDB writing.

TiDB will output binlog to Pump when DML statements are committed and DDL statements are completed.

### Statement Implementation Phase

DML statements include Insert/Replace, Update, Delete. We'll use Insert statements as an example here, other statements are similar. First, before inserting (not committed) after executing the insert statement, the new data will be recorded in the `binlog.TableMutation` structure.

This structure is kept in the transaction context structure of each Session link in `TxnState.mutations`. One table corresponds to one `TableMutation` object. `TableMutation` contains all the change data of this transaction to this table. Insert will format the current statement according to `RowID` + `Row-value` and add it to `TableMutation.InsertedRows`.

After all statements have been executed, `TxnState.mutations` contains the change data of all tables in the current transaction.

### Commit Stage

For DML, TiDB's transactions use the 2-phase-commit algorithm, and a transaction commit will be divided into the Prewrite stage and the Commit stage. Let's look at TiDB's specific behavior in these two stages.

#### Prewrite Binlog

In the `session.doCommit` function, TiDB will structure `binlog.PrewriteValue`:

This `PrewriteValue` contains all the row data related to this change. TiDB will fill in a type as `binlog.BinlogType_Prewrite` Binlog.

TiDB uses a business option `kv.BinlogInfo` to tie `BinlogInfo` to the current transaction object.

In `twoPhaseCommitter.execute`, when prewriting data to TiKV, it will call `twoPhaseCommitter.prewriteBinlog`. Here it will take out the related `binloginfo.BinlogInfo` and export the binlog `binlog.PrewriteValue` to Pump.

It's worth noting that in the prewrite stage, it's necessary to wait for the completion of writing prewrite binlog before continuing to do the next commit. This is to ensure that upon successful TiDB commit, Pump will at least receive the Prewrite binlog.

#### Commit Binlog

In `twoPhaseCommitter.execute`, after the transaction is committed, the transaction may be successfully committed or may fail. TiDB needs to inform Pump of this state:

If an error occurs, the type of binlog exported is `binlog.BinlogType_Rollback`. If successfully committed, the type of binlog exported is `binlog.BinlogType_Commit`.

It's worth noting that here WriteBinlog is done separately by initiating a goroutine in the Commit stage, and it's no longer necessary to wait to write binlog. This can save a little waiting time for commitment. There's no need to wait here because even if Pump can't receive this Commit binlog, after more than timeout time, Pump will confirm the submission status of the current transaction according to Prewrite binlog to TiKV.

## DDL Binlog

A DDL has the following states:

These states represent the state of a DDL job:

1. `JobStateNone`: Represents that the DDL job is still in the processing queue, TiDB has not yet started to do this DDL.
2. `JobStateRunning`: When DDL Owner starts to handle this task, it will set the status as `JobStateRunning`. After that, DDL will start to change, and TiDB's Schema may involve changes in multiple states. This will not change the state of DDL job, but only the state of Schema.
3. `JobStateDone`: When TiDB completes all of its Schema status changes, it will change Job's status to Done.
4. `JobStateSynced`: Whenever TiDB changes the status of schema once, it will need to be synchronized with other TiDBs in the cluster. After the status of Job is `JobStateDone`, after TiDB waits for all TiDB nodes to synchronize, it will modify the status to `JobStateSynced`.
5. `JobStateCancelling`: TiDB provides grammar `ADMIN CANCEL DDL JOBS job_ids` to cancel a DDL task that is being performed or not yet performed. When this order is successfully executed, the status of the DDL task will become `JobStateCancelling`.
6. `JobStateRollingback`: When DDL Owner discovers that Job's state became `JobStateCancelling`, it will change the status of job to `JobStateRollingback`, to show that the cancel request has been processed.
7. `JobStateRollbackDone`: In the process of doing cancel, it will also involve changes in Schema's state. It also needs to go through the synchronization of Schema. When the state rolls back, TiDB will set Job's state as `JobStateRollbackDone`.

For binlog, the binlog output mechanism of DDL is similar to DML statements. Only when the transaction submission stage is started will the binlog be written out. So for DDL, unlike DML which has a transaction concept, SQL's transaction does not affect DDL statements. However, in DDL, the above-mentioned Job's state change was submitted as a transaction (to ensure consistency of state). So in each state change, there will be a transaction corresponding to it, but for the intermediate states mentioned above, DDL will not write binlog out. Only in `JobStateRollbackDone` and `JobStateDone` states will TiDB consider that the DDL statement has been completed and will send binlog. Before sending, it will change Job's state from `JobStateDone` to `JobStateSynced`. This revision also involves a transaction submission.

`DdlQuery` will be set as the original DDL statement, `DdlJobId` will be set as DDL's task ID.

For the last submission of Job status, there will be two binlogs corresponding to it. There are several situations here:

1. If the transaction is submitted successfully, the types are `binlog.BinlogType_Prewrite` and `binlog.BinlogType_Commit`.
2. If the transaction submission fails, the types are `binlog.BinlogType_Prewrite` and `binlog.BinlogType_Rollback`.

So, for DDL binlog received by Pumps, if the type is `binlog.BinlogType_Rollback`, it should only be considered that the following states are legal:

1. `JobStateDone` (Because modification to `JobStateSynced` has not yet succeeded)
2. `JobStateRollbackDone`

If the type is `binlog.BinlogType_Commit`, it should only be considered that the following states are legal:

1. `JobStateSynced`
2. `JobStateRollbackDone`

When TiDB submits the last Job state, if the transaction submission fails, then TiDB Owner will try to continue to modify this Job until it succeeds. That is, for the same `DdlJobId`, there may be multiple binlogs in the follow-up until a `binlog.BinlogType_Commit` appears.

> Click to see more [TiDB source code reading series articles](https://cn.pingcap.com/blog/tag/tidb-source-code-reading/) 
