# TiDB Source Code Reading Series Article: INSERT Statement Types

This article will first introduce the classification of INSERT statements in TiDB, as well as the grammar and semantics of each statement, and then introduce the source code implementation of the five INSERT statements.

## Introduction

In a [previous article](https://cn.pingcap.com/blog/tidb-source-code-reading-4) "TiDB Source Code Reading Series Article (4): INSERT Statement Overview", we introduced the general process of INSERT statements. Why do we need to write another article about INSERT? Because in TiDB, simply inserting a piece of data is the simplest case and the most commonly used case. What's more complicated is how to handle various behaviors in the INSERT statement. For example, how to handle Unique Key conflicts: Should it return an error? Should it ignore the currently inserted data? Or should it overwrite existing data? Therefore, this article will continue to introduce INSERT statements in depth.

## Types of INSERT Statements

Broadly speaking, TiDB has the following six INSERT statements:

- `Basic INSERT`
- `INSERT IGNORE`
- `INSERT ON DUPLICATE KEY UPDATE`
- `INSERT IGNORE ON DUPLICATE KEY UPDATE`
- `REPLACE`
- `LOAD DATA`

These six statements are theoretically all INSERT statements.

The first one, `Basic INSERT`, is the most common INSERT statement with the syntax `INSERT INTO VALUES ()`. The semantics is to insert data, and if there is a unique constraint conflict (primary key conflict, unique index conflict), it returns an execution failure.

The second one, with syntax `INSERT IGNORE INTO VALUES ()`, ignores the current INSERT row and records a warning when INSERT encounters a unique constraint conflict. When the statement is executed, you can use `SHOW WARNINGS` to see which rows were not inserted.

The third one, with syntax `INSERT INTO VALUES () ON DUPLICATE KEY UPDATE`, updates the conflicting row after a conflict. If the updated row conflicts with another row in the table, an error is returned.

The fourth one is that in the last case, after the updated row conflicts with another row, it does not insert the row and displays a warning.

The fifth one, with syntax `REPLACE INTO VALUES ()`, deletes the conflicting row in the table after a conflict, and continues to try to insert the data. If it conflicts again, it continues to delete the conflicting data in the table until the data in the table does not conflict with the changed row.

The last one, with syntax `LOAD DATA INFILE INTO`, has the same semantics as `INSERT IGNORE` in that conflicts are ignored. The difference is that `LOAD DATA`'s role is to import a data file into the table, meaning the data comes from a CSV data file.

Since `INSERT IGNORE ON DUPLICATE KEY UPDATE` is just a special treatment on top of `INSERT ON DUPLICATE KEY UPDATE`, it will not be described separately but in the same section. `LOAD DATA`, due to its own particularity, will be left for other chapters.

## Basic INSERT

The biggest difference between several INSERT statements is at the execution level. Let's continue from [TiDB Source Code Reading Series Article (4): INSERT Statement Overview](https://cn.pingcap.com/blog/tidb-source-code-reading-4) to discuss the execution process. Students who don't remember the previous content can return to read the original article.

INSERT's execution logic is in [executor/insert.go](https://github.com/pingcap/tidb/blob/ab332eba2a04bc0a996aa72e36190c779768d0f1/executor/insert.go). In fact, the first four types of INSERT execution logic are all in this file. Let's talk about the most common `Basic INSERT`.

`InsertExec` is the executor of INSERT, which implements the Executor interface. The most important are the following three interfaces:

- Open: Some initialization
- Next: Perform writing operations  
- Close: Do some cleanup work

The most important and complex of these is the Next method. Based on whether data is obtained through a SELECT statement (`INSERT SELECT FROM`), the Next process is divided into [insertRows](https://github.com/pingcap/tidb/blob/ab332eba2a04bc0a996aa72e36190c779768d0f1/executor/insert_common.go#L180:24) and [insertRowsFromSelect](https://github.com/pingcap/tidb/blob/ab332eba2a04bc0a996aa72e36190c779768d0f1/executor/insert_common.go#L277:24) processes. Both processes will eventually enter the `exec` function to execute INSERT.

The first four INSERT statements are processed in the `exec` function, and the ordinary INSERT discussed in this section goes directly to [insertOneRow](https://github.com/pingcap/tidb/blob/5bdf34b9bba3fc4d3e50a773fa8e14d5fca166d5/executor/insert.go#L42:22).

Before talking about [insertOneRow](https://github.com/pingcap/tidb/blob/5bdf34b9bba3fc4d3e50a773fa8e14d5fca166d5/executor/insert.go#L42:22), let's look at SQL behavior.

It can be seen that for INSERT statements in TiDB, conflict testing is only done when the transaction is submitted, while MySQL tests when the statement is executed. The reason for this treatment is that TiDB has a layered structure with TiKV. For efficient execution, only read operations in transactions must obtain data from the storage engine, and all write operations are placed in the single TiDB instance's [memDbBuffer](https://github.com/pingcap/tidb/blob/ab332eba2a04bc0a996aa72e36190c779768d0f1/kv/memdb_buffer.go#L31). The transactions are only written to TiKV at once when committed.

In the implementation of [insertOneRow](https://github.com/pingcap/tidb/blob/5bdf34b9bba3fc4d3e50a773fa8e14d5fca166d5/executor/insert.go#L42:22), the [PresumeKeyNotExists](https://github.com/pingcap/tidb/blob/e28a81813cfd290296df32056d437ccd17f321fe/kv/kv.go#L23) option is set. If all INSERT operations detect no conflicts locally, it is assumed that the insertion will not conflict, and there is no need to check with TiKV whether conflicting data exists. These data are only marked as pending testing. In the final commit process, all data to be tested in the entire transaction will use the `BatchGet` interface for batch testing.

When all the data passes through [insertOneRow](https://github.com/pingcap/tidb/blob/5bdf34b9bba3fc4d3e50a773fa8e14d5fca166d5/executor/insert.go#L42:22) for insertion, the INSERT statement is basically complete. The remaining work is to set up return information such as lastInsertID and finally return the result to the client.

## INSERT IGNORE

The semantics of `INSERT IGNORE` has been introduced earlier. We introduced that ordinary INSERT checks for conflicts at commit time, so is `INSERT IGNORE` possible? The answer is no, because:

1. If `INSERT IGNORE` tested at commit time, the transaction module would need to know which rows need to be ignored and which should be reported and rolled back, which undoubtedly increases coupling between modules.
2. Users want to immediately get which rows are not included with `INSERT IGNORE` and immediately `SHOW WARNINGS` to see which rows are not actually written.

This requires implementing `INSERT IGNORE` to check data conflicts in real-time. An obvious approach would be to try to read the data that needs to be inserted. When a conflict is discovered, remember a warning and continue to the next row. However, when inserting multiple rows in one statement, it would be necessary to repeatedly read data from TiKV for testing. Obviously, this efficiency is not high. So, TiDB implemented [batchChecker](https://github.com/pingcap/tidb/blob/3c0bfc19b252c129f918ab645c5e7d34d0c3d154/executor/batch_checker.go#L43:6), with code in [executor/batch_checker.go](https://github.com/pingcap/tidb/blob/ab332eba2a04bc0a996aa72e36190c779768d0f1/executor/batch_checker.go).

In [batchChecker](https://github.com/pingcap/tidb/blob/3c0bfc19b252c129f918ab645c5e7d34d0c3d154/executor/batch_checker.go#L43:6), first take the data to be inserted and construct the unique keys that could conflict using [getKeysNeedCheck](https://github.com/pingcap/tidb/blob/3c0bfc19b252c129f918ab645c5e7d34d0c3d154/executor/batch_checker.go#L85:24) (TiDB implements unique constraints by constructing unique keys, as detailed in [Three Articles to Understand TiDB Technical Inside Story - Computing](https://cn.pingcap.com/blog/tidb-internal-2/)).

Then, use [BatchGetValues](https://github.com/pingcap/tidb/blob/c84a71d666b8732593e7a1f0ec3d9b730e50d7bf/kv/txn.go#L97:6) to read all the constructed keys at once and get a Key-Value map - anything that can be read represents conflicting data.

Finally, look up each upcoming data key in the [BatchGetValues](https://github.com/pingcap/tidb/blob/c84a71d666b8732593e7a1f0ec3d9b730e50d7bf/kv/txn.go#L97:6) results. If a conflicting row is found, construct warning information and move on to the next row. If no conflict is found, proceed with a safe INSERT. This part is implemented in [batchCheckAndInsert](https://github.com/pingcap/tidb/blob/ab332eba2a04bc0a996aa72e36190c779768d0f1/executor/insert_common.go#L490:24).

Similarly, after all data is inserted, return information is set and the execution results are returned to the client.

## INSERT ON DUPLICATE KEY UPDATE Statement

`INSERT ON DUPLICATE KEY UPDATE` is the most complicated of the INSERT statements. Its semantics essentially combines an INSERT and an UPDATE. Compared to other INSERT statements, the complexity is that UPDATE semantics can update a row to any legal state.

In the previous section, we introduced that TiDB used a batch method for special INSERT statements to implement conflict checking. When dealing with `INSERT ON DUPLICATE KEY UPDATE`, we adopted the same method, but due to the complexity of the semantics, the implementation steps are also much more complicated.

First, like `INSERT IGNORE`, construct the keys for the data to be inserted, use [BatchGetValues](https://github.com/pingcap/tidb/blob/c84a71d666b8732593e7a1f0ec3d9b730e50d7bf/kv/txn.go#L97:6) to read them once and get a Key-Value map. Then read out all the records in the Key correspondence table using [BatchGetValues](https://github.com/pingcap/tidb/blob/c84a71d666b8732593e7a1f0ec3d9b730e50d7bf/kv/txn.go#L97:6) - this data is prepared for future UPDATE in [initDupOldRowValue](https://github.com/pingcap/tidb/blob/3c0bfc19b252c129f918ab645c5e7d34d0c3d154/executor/batch_checker.go#L225:24).

Then, when doing conflict checking, if a conflict is encountered, first perform an UPDATE. We already introduced in the earlier Basic INSERT section that TiDB's INSERT only goes to TiKV to actually execute when committed. Similarly, UPDATE statements are actually executed by TiKV only when the transaction is committed. In this UPDATE, you may still encounter unique constraint conflicts. If encountered, return an error at this point. If the statement is `INSERT IGNORE ON DUPLICATE KEY UPDATE`, ignore this error and continue to the next row.

In the UPDATE step above, the following scenario also needs to be handled:

It can be seen that there is no data in the table in this SQL, and the INSERT in the second statement cannot read potentially conflicting data, but the two rows of data to be inserted by INSERT itself conflict. The correct execution here should be that the first 1 is inserted normally, when the second 1 is inserted a conflict is found, and the first 1 is updated. At this time, the following treatment is required: The Key-Value corresponding to UPDATE data from the first step is deleted from the Key-Value map, and the data from UPDATE is constructed as unique constraint keys and values based on its table information. Read back into the first step's Key-Value map for subsequent data conflict checking. The implementation of this detail is in [fillBackKeys](https://github.com/pingcap/tidb/blob/2fba9931c7ffbb6dd939d5b890508eaa21281b4f/executor/batch_checker.go#L232). This scenario also appears in other INSERT statements like `INSERT IGNORE`, `REPLACE`, `LOAD DATA`. The reason for introducing it here is because `INSERT ON DUPLICATE KEY UPDATE` most completely demonstrates the various features of `batchChecker`.

Finally, after all data is inserted/updated, return information is set and the execution results are returned to the client.

## REPLACE

Although the REPLACE statement looks like an independent class of DML, actually observing the grammar, it just replaces INSERT with REPLACE compared to `Basic INSERT`. Unlike all the INSERT statements introduced before, the REPLACE statement is a one-to-many statement. To briefly explain, if a general INSERT statement needs to INSERT a certain row, then when it encounters a unique constraint conflict, the following treatments will appear:

- Give up insertion and return error: `Basic INSERT`
- Give up insertion, no error: `INSERT IGNORE`  
- Give up insertion and change to updating the conflicting row, if the updated value conflicts again:
  - Return error: `INSERT ON DUPLICATE KEY UPDATE`
  - No error: `INSERT IGNORE ON DUPLICATE KEY UPDATE`

They all handle different treatments when a row of data conflicts with a row in the table. However, REPLACE has different semantics - it will delete all conflicts encountered until the data can be inserted without conflicts. If there are 5 unique indexes in the table, there may be 5 rows that conflict with the row waiting to be inserted. Then REPLACE will delete these 5 rows at once and insert itself.

After understanding the particularity of the REPLACE statement, we can more easily understand its specific implementation.

Similar to INSERT statements, the main operative part of REPLACE is also in its Next method, which is different from INSERT. [insertRowsFromSelect](https://github.com/pingcap/tidb/blob/ab332eba2a04bc0a996aa72e36190c779768d0f1/executor/insert_common.go#L277:24) and [insertRows](https://github.com/pingcap/tidb/blob/ab332eba2a04bc0a996aa72e36190c779768d0f1/executor/insert_common.go#L180:24) pass through [ReplaceExec](https://github.com/pingcap/tidb/blob/f6dbad0f5c3cc42cafdfa00275abbd2197b8376b/executor/replace.go#L27)'s own [exec](https://github.com/pingcap/tidb/blob/f6dbad0f5c3cc42cafdfa00275abbd2197b8376b/executor/replace.go#L160) method. In [exec](https://github.com/pingcap/tidb/blob/f6dbad0f5c3cc42cafdfa00275abbd2197b8376b/executor/replace.go#L160) it calls [replaceRow](https://github.com/pingcap/tidb/blob/f6dbad0f5c3cc42cafdfa00275abbd2197b8376b/executor/replace.go#L95), which also used [batchChecker](https://github.com/pingcap/tidb/blob/3c0bfc19b252c129f918ab645c5e7d34d0c3d154/executor/batch_checker.go#L43:6)'s batch conflict detection. The difference from INSERT is that here, all detected conflicts will be deleted, and finally the row to be inserted will be written.

## Conclusion

INSERT statements are the most complex and feature-rich of all DML statements. There are statements like `INSERT ON DUPLICATE UPDATE` that can execute both INSERT and UPDATE, and statements like REPLACE that can affect many rows of data. The INSERT statement itself can be connected to a SELECT statement as input for the data to be inserted, so it is affected by the planner again (for the planner part, see the relevant source code reading articles: [(7) Rule-based Optimization](https://cn.pingcap.com/blog/tidb-source-code-reading-7) and [(8) Cost-based Optimization](https://cn.pingcap.com/blog/tidb-source-code-reading-8)). Understanding TiDB's implementation of various INSERT statements can help readers better use these statements in the future and choose the most reasonable and efficient statements according to their characteristics. Additionally, readers interested in contributing code to TiDB can understand this part's implementation more quickly through this article.

> Click to see more [TiDB Source Code Reading Series Articles](https://cn.pingcap.com/blog/tag/tidb-source-code-reading/) 
