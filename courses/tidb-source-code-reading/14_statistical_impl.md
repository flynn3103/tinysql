# TiDB Source Code Reading Series Article: Statistical Information Implementation

This article introduces the direct map and the data structure of Count-Min(CM)Sketch, and then explains how TiDB has implemented statistical information query, collection, and update.

## Introduction

In [Statistical Information (Part 1)](https://cn.pingcap.com/blog/tidb-source-code-reading-12/), we introduced the basic concepts of statistical information, TiDB's statistical information collection/update mechanism, and how to use statistical information to estimate calculation costs. This article will explore the TiDB source code implementation of these principles.

We'll first examine the direct map and Count-Min(CM)Sketch data structures, then delve into how TiDB implements statistical information querying, collection, and updates.

## Data Structure Definitions

### Histogram Definition
The histogram definition can be found in [histogram.go](https://github.com/lamxTyler/tidb/blob/source-code/statistics/histogram.go#L40). Notably, we use Chunk (introduced in [TiDB Source Code Reading Series Article: Chunk and Execution Framework](https://cn.pingcap.com/blog/tidb-source-code-reading-10/)) to store bucket upper and lower bounds, which reduces memory allocation costs compared to using Datum.

### CM Sketch Definition
The CM Sketch definition can be found in [cmsketch.go](https://github.com/lamxTyler/tidb/blob/source-code/statistics/cmsketch.go#L31). It's relatively simple, containing:
- The core two-dimensional array `table` for CM Sketch
- Stored depth and width
- Total number of inserted values (though these can be derived directly from `table`)

Statistical information for columns and indexes are stored separately using [Column](https://github.com/lamxTyler/tidb/blob/source-code/statistics/histogram.go#L699) and [Index](https://github.com/lamxTyler/tidb/blob/source-code/statistics/histogram.go#L773) structures respectively, mainly containing histograms, CM Sketch, and other related information.

## Statistical Information Creation

When executing the ANALYZE statement, TiDB collects both histogram and CM Sketch information. The process begins by splitting the analysis of columns and indexes into different tasks in [builder.go](https://github.com/lamxTyler/tidb/blob/source-code/plan/planbuilder.go#L609), then pushing these tasks down to TiKV for execution via [analyze.go](https://github.com/lamxTyler/tidb/blob/source-code/executor/analyze.go#L114). Since TiKV is included in TiDB, we'll focus on explaining the histogram creation process in the TiDB code.

### Histogram Creation

As mentioned in Statistical Information (Part 1), histogram creation involves first sampling data and then building the histogram.

The [collect](https://github.com/lamxTyler/tidb/blob/source-code/statistics/sample.go#L113) function implements the reservoir sampling algorithm to generate a uniform sample collection. Since its principles and code are relatively straightforward, we won't delve into the details here.

After sampling is complete, [BuildColumn](https://github.com/lamxTyler/tidb/blob/source-code/statistics/builder.go#L97) implements histogram creation. It first sorts the samples to determine bucket heights, then processes each value V in order:

1. If V equals the previous value, place V in the same bucket regardless of whether the bucket is full, ensuring each value exists in only one bucket.

2. If V is not equal to the previous value, check if the current bucket is full:
   - If not full, place it directly in the current bucket and use [updateLastBucket](https://github.com/lamxTyler/tidb/blob/source-code/statistics/builder.go#L146) to update the bucket's upper boundary and depth.
   
3. Otherwise, use [AppendBucket](https://github.com/lamxTyler/tidb/blob/source-code/statistics/builder.go#L151) to create a new bucket.

### Index Histogram

For index histograms, we use [SortedBuilder](https://github.com/alivxxx/tidb/blob/source-code/statistics/builder.go#L24) to maintain the intermediate state during construction. Since we can't know the data volume in advance, we can't predetermine bucket depths. However, since index data is already ordered, we initially set each bucket's depth to 1 in [NewSortedBuilder](https://github.com/alivxxx/tidb/blob/source-code/statistics/builder.go#L38).

For each piece of data, [Iterate](https://github.com/alivxxx/tidb/blob/source-code/statistics/builder.go#L50) inserts data using a method similar to histogram creation. If at some point the required number of buckets exceeds the current bucket depth, [mergeBucket](https://github.com/alivxxx/tidb/blob/source-code/statistics/builder.go#L74) combines each previous two buckets into one, doubles the bucket depth, and continues insertion.

After collecting histograms established on each Region, [MergeHistogram](https://github.com/alivxxx/tidb/blob/source-code/statistics/histogram.go#L609) combines the histograms from each region:

1. To ensure each value exists in only one bucket, we handle boundary bucket issues:
   - If two adjacent buckets are [equal](https://github.com/alivxxx/tidb/blob/source-code/statistics/histogram.go#L623), merge them first

2. Before merging, [adjust](https://github.com/alivxxx/tidb/blob/source-code/statistics/histogram.go#L642) the average bucket depths of both histograms to be roughly equal

3. If the number of buckets after merging exceeds the limit, merge adjacent buckets two by two using [one](https://github.com/alivxxx/tidb/blob/source-code/statistics/histogram.go#L653)

## Statistical Information Maintenance

In Statistical Information (Part 1), we introduced how TiDB updates histograms and CM Sketch. The CM Sketch update is relatively simple and won't be covered here. This section focuses on how TiDB collects feedback and maintains histograms.

### Feedback Information Collection

As mentioned in Part 1, to avoid assuming even error distribution across buckets, we need to collect feedback information for each bucket by dividing query ranges according to histogram bucket boundaries.

In [SplitRange](https://github.com/alivxxx/tidb/blob/source-code/statistics/histogram.go#L511), we split query ranges according to the histogram. Since current histogram buckets contain both upper and lower bounds, we split only by upper bounds for convenience. The range for bucket i is treated as `(bucket i-1's upper bound, bucket i's upper bound]`. For the last bucket, the upper bound is treated as infinite.

For example, with a histogram containing 3 buckets with ranges [2,5], [8,8], [10,13], and a query range (3,20), we split into (3,5], (5,8], (8,20].

After splitting query ranges, they're stored in [QueryFeedback](https://github.com/alivxxx/tidb/blob/source-code/statistics/feedback.go#L49). When results from each region return, the [Update](https://github.com/alivxxx/tidb/blob/source-code/statistics/feedback.go#L165) function updates the number of keys in each range. This function requires two parameters:
- The start key scanned on each Region
- The key count outputs for each scan range on the Region

To update the number of keys in each QueryFeedback range, we need to know which query range corresponds to each output count. Since coprocessor output counts are continuous and the same value only corresponds to one range, we only need to know the range corresponding to the first output count - that is, just the scan's start key.

### Histogram Updates

After collecting QueryFeedback, we use [UpdateHistogram](https://github.com/alivxxx/tidb/blob/source-code/statistics/feedback.go#L536) to update the histogram. This process involves both splitting and merging operations.

In [splitBuckets](https://github.com/alivxxx/tidb/blob/source-code/statistics/feedback.go#L503), we implement histogram splitting:

Note that buckets that are too small won't be split, and if a post-split bucket would be too small, it won't be generated.

After bucket splitting is complete, we use [mergeBuckets](https://github.com/alivxxx/tidb/blob/source-code/statistics/feedback.go#L467) to merge buckets:

1. During splitting, we record whether each bucket is new. For pre-existing buckets, use [getBucketScore](https://github.com/alivxxx/tidb/blob/source-code/statistics/feedback.go#L476) to calculate the error after merging:
   - If the ratio of the first bucket to the combined bucket is r, the merge error is |first bucket's pre-merge height - r * combined bucket height| / first bucket's pre-merge height

2. Sort the merge errors for each bucket

3. Finally, merge the required buckets in order from lowest to highest error

## Use of Statistical Information

In query statements, we often use filtering conditions. The main function of statistical information estimation is to estimate the number of data rows after these filtering conditions to optimize the execution plan.

Since list queries are relatively simple, we won't repeat the details here. The code implementation follows the principles outlined in Statistical Information (Part 1). For those interested, refer to [histor.go/lessRowCount](https://github.com/alivxxx/tidb/blob/source-code/statistics/histogram.go#L408) and [cmsketch.go/queryValue](https://github.com/alivxxx/tidb/blob/source-code/statistics/cmsketch.go#L69).

### Multiple Queries

As mentioned in Part 1, [Selectivity](https://github.com/alivxxx/tidb/blob/source-code/statistics/selectivity.go#L148) is the most important interface the statistics module provides to the optimizer for handling multiple queries. One of Selectivity's most important tasks is dividing all query conditions into as few groups as possible, so conditions in each group can be estimated using statistical information from a certain column or index, minimizing independence assumptions.

Note that we divide separate statistical information into 3 categories:
- [indexType](https://github.com/alivxxx/tidb/blob/source-code/statistics/selectivity.go#L42): Index
- [pkType](https://github.com/alivxxx/tidb/blob/source-code/statistics/selectivity.go#L43): Integer type primary key
- [colType](https://github.com/alivxxx/tidb/blob/source-code/statistics/selectivity.go#L44): Ordinary column type

If a condition can be covered by multiple types of statistical information simultaneously, we prefer to choose pkType or indexType.

In Selectivity, there are the following steps:

1. [getMaskAndRange](https://github.com/alivxxx/tidb/blob/source-code/statistics/selectivity.go#L230) calculates which filtering conditions can be covered for each column and index:
   - Uses an int64 as a bitset
   - Sets 1 at positions where filtering conditions can be covered by the column

2. Next, [getUsableSetsByGreedy](https://github.com/alivxxx/tidb/blob/source-code/statistics/selectivity.go#L258) chooses as few bitsets as possible to cover as many filtering conditions as possible:
   - From unused bitsets, choose one that can cover the most uncovered filtering conditions
   - When able to cover the same number of conditions, prefer pkType or indexType

3. Use the method mentioned in Part 1 to estimate statistical information on each column and index, then combine them using independence assumptions for the final result

## Summary

Collection and maintenance of statistical information is a core database function. For cost-based query optimizers, statistical information accuracy directly affects query efficiency. In distributed databases, while collecting statistical information is similar to standalone databases, maintaining statistical information presents greater challenges, such as how to maintain accuracy and timeliness when multiple nodes are updated.

For dynamic histogram updates, the industry generally has two approaches:

1. Update corresponding bucket depths for each addition/deletion:
   - When a bucket becomes too deep, usually split it by bucket width
   - Difficult to accurately determine split points, leading to errors

2. Use actual query numbers to adjust histograms:
   - Assume all bucket contribution errors are even
   - Use continuous value assumptions to adjust all involved buckets
   - Even error assumption often causes problems
   - For example, when newly inserted values exceed the histogram's maximum value, attributing their error across the histogram causes inaccuracies

Currently, TiDB's statistical information is primarily based on separate statistics. To reduce independence assumptions, TiDB will explore collecting and maintaining multi-column statistics to provide more accurate statistical information for optimizers.

[Click to see more TiDB source code reading series articles](https://cn.pingcap.com/blog/tag/tidb-source-code-reading/) 
