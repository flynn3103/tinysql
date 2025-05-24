# TiDB Source Code Reading Series Article: Prepared Statement Implementation

In [TiDB source code reading series article (3): SQL's lifetime](https://cn.pingcap.com/blog/tidb-source-code-reading-3/), we introduced TiDB's most common request processing process of `Command -- COM_QUERY` when receiving client request packages. In this article, we will introduce another commonly used process in TiDB: `Command -- Prepare/Execute` requests.

## Prepare/Execute Statement

Let's briefly review the client's use of the Prepare request process:

1. The client initiates a Prepare command with an SQL statement containing "?" parameter placeholders, and upon success receives a `stmtID`.
2. When executing SQL specifically, the client uses the previously returned `stmtID` and request parameters to initiate an Execute command.
3. When the Prepare statement is no longer needed, close the Prepare statement corresponding to the `stmtID`.

Compared to ordinary requests, the benefits of Prepare are:

* Reduce the burden of each execution through Parser, because in many scenarios, online SQLs are mostly the same content with different parameters. Through Prepare, you can prepare SQL with placeholders once and just fill in the parameters to execute, achieving "parse once, use many times".
* After enabling PreparePlanCache, you can achieve "optimize once, use many times", without repeating the logical and physical optimization process.
* Less network transmission, because multiple executions only transmit parameter parts and return results in Binary protocol.
* Because parameters are filled at execution time, it can prevent SQL injection risks.
* Certain features such as serverSideCursor need to be used through Prepare statements.

TiDB, like the [MySQL Protocol](https://dev.mysql.com/doc/refman/5.7/en/sql-prepared-statements.html), has two ways to use Prepare/Execute:

* Binary Protocol: Using `COM_STMT_PREPARE`, `COM_STMT_EXECUTE`, `COM_STMT_CLOSE` commands and obtaining results through Binary protocol, which is currently the commonly used method in various applications.
* Text Protocol: Using `COM_QUERY` with `PREPARE`, `EXECUTE`, `DEALLOCATE PREPARE` commands and text protocol to obtain results. This is not as efficient as the binary protocol and is mostly used for non-program call scenarios, such as manual execution in the MySQL client.

Below we'll mainly look at TiDB's processing with the Binary protocol. The processing of text protocols is similar, and we'll briefly introduce their differences later.

## `COM_STMT_PREPARE`

When the client initiates `COM_STMT_PREPARE`, TiDB will enter `clientConn#handleStmtPrepare` after receiving it. This function calls `TiDBContext#Prepare` to perform actual Prepare operations and return [results](https://dev.mysql.com/doc/dev/mysql-server/latest/) to the client. The actual Prepare processing is mainly completed in `session#PrepareStmt` and `PrepareExec`:

1. Call Parser to complete text to AST conversion, see [TiDB source code reading series article (5): Implementation of TiDB SQL Parser](https://cn.pingcap.com/blog/tidb-source-code-reading-5/).
2. Use `paramMarkerExtractor` visitor to extract "?" expressions from AST and sort them by appearance position (offset). This Slice will be used later to quickly locate and replace "?" placeholders.
3. Check if the number of parameters exceeds Uint16 maximum value (this is a [protocol restriction](https://dev.mysql.com/doc/dev/mysql-server/latest/), only 2 bytes for parameters).
4. Perform Preprocess and create LogicPlan, see previous [introduction to logical optimization](https://cn.pingcap.com/blog/tidb-source-code-reading-7/). LogicPlan is generated here mainly to obtain and check information needed in the Prepare response.
5. Generate `stmtID` - an incrementing int in the current session.
6. Save `stmtID` to `ast.Prepared` (mapping of AST, parameter type info, schema version, whether to use `PreparedPlanCache` flag) in `SessionVars#PreparedStmts` for use by Execute.
7. Save `stmtID` to `TiDBStatement` (mapping of `stmtID`, parameter count, SQL return column type info, `sendLongData` pre-`BoundParams`) in `TiDBContext#stmts`.

After processing completes, the client receives and holds `stmtID` and parameter type information, return column type information, which can be used later during execution to find previously Prepared information through mappings saved in steps 6 or 7.

## `COM_STMT_EXECUTE`

After Prepare succeeds, the client sends a `COM_STMT_EXECUTE` command to request execution. TiDB enters `clientConn#handleStmtExecute`, which first uses stmtID to get the previously saved `TiDBStatement` from `TiDBContext#stmts` and analyzes whether to use `userCursor` and request parameter information, then calls the corresponding `TiDBStatement` Execute to perform actual Execute logic:

1. Generate `ast.ExecuteStmt` and call `planer.Optimize` to generate `plancore.Execute`. Unlike the general optimization process, it will execute `Execute#OptimizePreparedPlan`.
2. Use `stmtID` to get the `ast.Prepared` information from the Prepare stage through `SessionVars#PreparedStmts`.
3. Use `prepared.Params` prepared in step 2 of the previous section to quickly find and fill parameter values; also save parameters to `sessionVars.PreparedParams` for supporting `PreparePlanCache` delayed parameter retrieval.
4. Check if there are schema changes between Prepare and Execute, and if so, repreprocess.
5. Call `Execute#getPhysicalPlan` to obtain the physical plan. The implementation will first look for cached plans based on whether PreparedPlanCache is used. We'll introduce this later in this article.
6. When PreparedPlanCache is not enabled or cache is enabled but not hit, perform normal Optimize on AST.

After getting PhysicalPlan, proceed with normal [execution](https://zhuanlan.zhihu.com/p/35134962).

## `COM_STMT_CLOSE`

When the client no longer needs to execute a previous Prepared statement, it can use `COM_STMT_CLOSE` to release server resources. After receiving this, TiDB enters `clientConn#handleStmtClose`, uses `stmtID` to find the corresponding `TiDBStatement` in `TiDBContext#stmts` and executes Close's cleanup before cleaning `TiDBContext#stmts` and `SessionVars#PrepareStmts`. However, looking at the code we see the former is cleaned directly while the latter is not deleted but added to `RetryInfo#DroppedPreparedStmtIDs` waiting for the current transaction to commit or rollback. The reason for delayed deletion of `SessionVars#PrepareStmts` is that TiDB will decide whether to retry the transaction based on configuration when encountering conflicts during transaction commit. The statements involved in retrying may only be Execute and Deallocate. TiDB currently uses `stmtID` to find prepared statements until transaction execution completes.

## Other `COM_STMT` Commands

In addition to the 3 introduced above, several other `COM_STMT` commands `COM_STMT_SEND_LONG_DATA`, `COM_STMT_FETCH`, `COM_STMT_RESET` are also used in Prepare.

### `COM_STMT_SEND_LONG_DATA`

In some scenarios our SQL parameters are `TEXT`, `TINYTEXT`, `MEDIUMTEXT`, `LONGTEXT` and `BLOB`, `TINYBLOB`, `MEDIUMBLOB`, `LONGBLOB` types. The client usually doesn't bring large parameters in an Execute, but sends them separately via [`COM_SEND_LONG_DATA`](https://dev.mysql.com/doc/dev/mysql-server/latest/) to TiDB in advance, then performs Execute.

TiDB's processing is in `client#handleStmtSendLongData`, using `stmtID` to find `TiDBStatement` in `TiDBContext#stmts` and place advance `paramID` corresponding parameter information, adding parameters to `boundParams` (so the client can actually send data multiple times and add to a parameter). Execute will use `stmt.BoundParams()` to obtain parameters passed in advance and execute with Execute command parameters, resetting `boundParams` after each execution.

### `COM_STMT_FETCH`

After normal Execute execution, TiDB will continue returning results to the client, with return rate controlled by `max_chunk_size` (see [TiDB source code reading series article (10): Chunk and executor framework introduction](https://cn.pingcap.com/blog/tidb-source-code-reading-10/)). But the actual result set may be very large. The client is limited by resources (usually memory) and cannot process so much data at once, so it hopes the server will return in batches. [`COM_STMT_FETCH`](https://dev.mysql.com/doc/dev/mysql-server/latest/) solves this problem.

Its use must first cooperate with `COM_STMT_EXECUTE` (i.e. must use Prepared statements). `handleStmtExeucte` has a flag in the request protocol to use cursor. Execute does not execute immediately after completing the plan and getting results but slowly stores them in `TiDBStatement`, and immediately returns a package to the client with listing information and [`ServerStatusCursorExists`](https://dev.mysql.com/doc/internals/en/status-flags.html) flag. This logic can be seen in `handleStmtExecute`.

When the client sees `ServerStatusCursorExists`, it will use `COM_STMT_FETCH` to pull specified fetchSize results from TiDB. In `connClient#handleStmtFetch` it will find `TiDBStatement` through session then find the previously cached result set, and start to actually call the executor's Next to obtain data meeting fetchSize and return to client. If the executor exceeds fetchSize once, it will only return fetchSize size data and keep remaining data for next time. Finally mark the last result set return with [`ServerStatusLastRowSend`](https://dev.mysql.com/doc/internals/en/status-flags.html) flag to inform the client there is no more data.

### `COM_STMT_RESET`

Mainly used for client actively replacing `COM_SEND_LONG_DATA` sent data. Normal `COM_STMT_EXECUTE` will automatically reset afterwards, mainly for when client wants to actively discard previous data, since `COM_STMT_SEND_LONG_DATA` is an additional operation requiring certain client scenarios to voluntarily give up pre-preserved parameters. This logic is mainly in `connClient#handleStmtReset`.

## Prepared Plan Cache

Through the front analysis process we saw that AST conversion was completed in Prepare, and subsequent Execute will use `stmtID` to find previous AST to skip Parse SQL costs each time. If Prepare Plan Cache is enabled, the last PhysicalPlan result can be further reused in Execute execution, eliminating query optimization process costs.

TiDB can enable Prepare Plan Cache by modifying the configuration file. After enabling, each new Session will initialize `SimpleLRUCache` type `preparedPlanCache` to save Plan results. The cache key is `pstmtPlanCacheKey` (connected by current DB, ID, `statementID`, `schemaVersion`, `snapshotTs`, `sqlMode`, `timezone` - so these elements must match the last cache for plan cache), and LRU should be done according to configuration cache size and memory size.

Execute's processing logic in `PrepareExec` besides checking if `PreparePlanCache` is enabled will also judge if the current statement can use `PreparePlanCache`.

1. Only `SELECT`, `INSERT`, `UPDATE`, `DELETE` may use `PreparedPlanCache`.
2. Further check through `cacheableChecker` visitor whether AST contains variable expressions, query, "order by?", "limit?,?" and UnCacheableFunctions function calls and other situations where PlanCache cannot be used.

If checks pass, `Execute#getPhysicalPlan` will build cache key with current environment `preparePlanCache`.

### Cache Miss

Let's first look at when Cache is not hit. When no hit is found it will use `stmtID` to find AST and implement Optimize, but different from normal Optimize implementation. For Cache's Plan, "?" placeholders need delayed value lookup, converting placeholders to functions to make Plan and Cache, then getting functions from Cache after execution to obtain actual execution parameters from specific execution context.

Recall that LogicPlan construction process will use `expressionRewriter` to convert AST into various `expression.Expression` types. Usually `ParamMarkerExpr` will be rewritten as Constant type expression, but if stmt supports Cache, it will be rewritten as Constant with special `DeferredExpr` pointing to a `GetParam` function expression. This function will actually get parameters saved from Execute to `sessionVars.PreparedParams`. This way Plan and Cache a parameter-independent Plan, then fill parameters when actually executing.

The new plan will later be saved to `preparedPlanCache` for subsequent use.

### Cache Hit

Back to `getPhysicalPlan`, if Cache hits the plan after getting it, we need to rebuild the plan because the saved plan has `GetParam` function expressions, and after getting it again current parameter values have changed. We need to recorrect ranges according to current Execute parameters. This logic is in `Execute#rebuildRange`. After that is normal execution process.

## Prepared by Text Protocol

We mainly introduced the binary protocol Prepared execution process above. There is also a way to execute through text protocol.

Client can send via `COM_QUERY`:

```sql
PREPARE stmt FROM 'SELECT * FROM t WHERE id = ?';
EXECUTE stmt USING @a;
DEALLOCATE PREPARE stmt;
```

To Prepare, TiDB will go through normal [text query process](https://zhuanlan.zhihu.com/p/35134962), converting SQL to Prepare, Execute, Deallocate Plans, and eventually converting to same executors as binary protocol: `PrepareExec`, `ExecuteExec`, `DealocateExec`.

## Summary

Prepared is one of the effective means to improve SQL execution efficiency. Familiarity with TiDB's Prepared implementation can help readers better use Prepared in the future. Additionally, readers interested in contributing code to TiDB can more quickly understand this part's implementation through this article.

> Click to see more [TiDB source code reading series articles](https://cn.pingcap.com/blog/tag/tidb-source-code-reading/) 
