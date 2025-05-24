# TiDB Source Code Reading Series Article: Logical Optimization Rules (Part 2)

This article introduces optimization rules such as aggregation elimination, outer join elimination, and subquery optimization.

## Introduction

In [TiDB source code reading series article (7): Rule-based Optimization](https://cn.pingcap.com/blog/tidb-source-code-reading-7/), we introduced several logical optimization rules in TiDB, including column pruning, max/min elimination, projection elimination, predicate pushdown, and node attribute construction. This article continues to introduce more optimization rules: aggregation elimination, outer join elimination, and subquery optimization.

## Aggregation Elimination

Aggregation Elimination checks whether columns used in SQL `Group By` statements have unique attributes. If satisfied, it will replace the `LogicalAggregation` operator in the plan with a `LogicalProjection` operator. The logic here is that when the aggregation function is grouped by one or more columns with unique attributes, each row of the lower calculation output is a separate grouping. At this time, the aggregation function can be expanded into a specific parameter column or a common function expression with a parameter list. The specific code is implemented in [`rule_aggregation_elimination.go`](https://github.com/eurekaka/tidb/blob/logical_rules_reading/planner/core/rule_aggregation_elimination.go).

Here are some specific examples:

Example 1:
The following query can expand the aggregation function into a column:

```sql
select max(b) from t group by a;  -- where 'a' has a unique attribute
```

Can be rewritten as:

```sql
select b from t;
```

Example 2:
The following query can expand the aggregation function to include the built-in function of the parameter column:

```sql
select count(a) from t group by id;  -- where 'id' has a unique attribute
```

Can be rewritten as:

```sql
select if(isnull(a), 0, 1) from t;
```

Note that this can be further optimized: if column `a` has a `Not Null` attribute, then `if(isnull(a), 0, 1)` can be directly replaced with constant 1.

Additionally, for most aggregation functions, the parameter type and return result type are generally different. Therefore, when the aggregation function is expanded, a cast function is usually constructed on the parameter column for type conversion. The expanded expression will be preserved in the `LogicalProjection` operator that replaces the `LogicalAggregation` operator.

In this optimization process, one crucial aspect is determining whether the `Group By` columns satisfy the uniqueness attribute, especially when the lower nodes of the aggregation are not `DataSource`. As mentioned in [(7) Rule-based Optimization](https://cn.pingcap.com/blog/tidb-source-code-reading-7/) in the "Node Attribute Construction" section, each operator in the execution plan maintains information about which columns satisfy the unique attribute in its output. Therefore, during aggregation elimination, we can check this information saved by the lower operators and combine it with the `Group By` columns to determine whether the current aggregation can be eliminated.

## Outer Join Elimination

Different from the "Converting Outer Join to Inner Join" mentioned in [(7) Rule-based Optimization](https://cn.pingcap.com/blog/tidb-source-code-reading-7/), outer join elimination here refers to removing the entire join operation from the query.

Outer join elimination requires certain conditions to be met:

* Condition 1: `LogicalJoin`'s parent only uses `LogicalJoin`'s outer plan
* Condition 2:
  * Condition 2.1: `LogicalJoin`'s join key meets the unique attribute in the output of inner plan
  * Condition 2.2: `LogicalJoin`'s parent only needs the outer plan's output

Conditions 1 and 2 must both be met, but only one of conditions 2.1 and 2.2 needs to be satisfied.

Example meeting conditions 1 and 2.1:

```sql
select t1.* from t1 left join t2 on t1.a = t2.a;  -- where t2.a is unique
```

Can be rewritten as:

```sql
select * from t1;
```

Example meeting conditions 1 and 2.2:

```sql
select t1.* from t1 left join t2 on t1.a = t2.a where t2.b > 1;
```

Can be rewritten as:

```sql
select * from t1;
```

The specific implementation of this optimization is in [rule_join_elimination.go](https://github.com/eurekaka/tidb/blob/logical_rules_reading/planner/core/rule_join_elimination.go).

## Subquery Optimization

Subqueries are divided into non-correlated subqueries and correlated subqueries. For example:

```sql
select * from t1 where t1.a > (select max(t2.a) from t2);  -- non-correlated
select * from t1 where t1.a > (select max(t2.a) from t2 where t2.b = t1.b);  -- correlated
```

For non-correlated queries, TiDB performs two types of operations in the `expressionRewriter` logic:

1. **Subquery Materialization**
   - Directly executes the subquery to obtain the result
   - Uses this result to rewrite the expression that originally included the subquery
   - For example, if the result returned is "1", the entire query will be rewritten as:
   ```sql
   select * from t1 where t1.a > 1;
   ```
   Detailed code logic can be found in [expression_rewriter.go](https://github.com/eurekaka/tidb/blob/logical_rules_reading/planner/core/expression_rewriter.go) in the [handleScalarSubquery](https://github.com/eurekaka/tidb/blob/logical_rules_reading/planner/core/expression_rewriter.go#L685) and [handleExistSubquery](https://github.com/eurekaka/tidb/blob/logical_rules_reading/planner/core/expression_rewriter.go#L535) functions.

2. **Converting Subquery to Join**
   For queries containing IN (subquery), such as:
   ```sql
   select * from t1 where t1.a in (select a from t2);
   ```
   Will be rewritten as:
   ```sql
   select t1.* from t1 inner join (select a, count(*) cnt from t2 group by a) t2 on t1.a = t2.a;
   ```
   If `t2.a` satisfies the unique attribute, according to the aggregation elimination rules introduced above, the query will be further rewritten as:
   ```sql
   select t1.* from t1 inner join t2 on t1.a = t2.a;
   ```

For correlated queries, TiDB will translate the entire expression containing correlated queries into a `LogicalApply` operator in the `expressionRewriter`. The specific code logic is implemented in [rule_decorrelate.go](https://github.com/eurekaka/tidb/blob/logical_rules_reading/planner/core/rule_decorrelate.go).

## Summary

This is the second article based on rule optimization. Later we will introduce more logical optimization rules: aggregation pushdown, TopN pushdown, and Join Reorder.

> Click to see more [TiDB source code reading series articles](https://cn.pingcap.com/blog/tag/tidb-source-code-reading/) 
