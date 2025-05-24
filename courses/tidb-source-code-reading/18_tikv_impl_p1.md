# TiDB Source Code Reading Series Article: TiKV Client Implementation

This article will detail several specific issues that tikv-client needs to solve during data reading and writing.

Throughout the implementation of SQL, the main steps of Parser, Optimizer, Executor, DistSQL are required. The final data reading and writing is done through tikv-client and TiKV cluster communication.

In order to complete the task of data literacy, tikv-client needs to solve the following specific problems:

1. How to locate to the TiKV address where a key or key range is located?
2. How to establish and maintain the connection with tikv-server?
3. How to send RPC request?
4. How to deal with various errors?
5. How to achieve distributed reading of data from multiple TiKV nodes?
6. How to achieve 2PC transactions?

We will answer the above questions one by one, of which 5 and 6 will be introduced in the next article.

## How to locate tikv-server where key is located

We need to review an important concept introduced in [Three articles to understand TiDB technology inside story — Storage](https://cn.pingcap.com/blog/tidb-internal-1/): Region.

The data distribution of TiDB is in Region, a Region contains data in a range, usually 96MB size, and the meta information of Region includes two attributes, StartKey and EndKey. When a key >= StartKey & key < EndKey, we knew the region where the key is located, and then we can go to this address to find the TiKV address where the region is located and read this key data.

Getting the region where key is located is done by sending a request to PD. PD client realized such an interface:

[GetRegion(ctx context.Context, key []byte) (*metapb.Region, *metapb.Peer, error)](https://github.com/pingcap/tidb/blob/v2.1.0-rc.1/vendor/github.com/pingcap/pd/pd-client/client.go#L49)

By calling this interface, we can locate the Region where this key is located.

If we need to get multiple Regions in a range, we will start from the StartKey of this range, call the `GetRegion` interface multiple times, using each returned Region's EndKey as the StartKey for the next request, until the returned Region's EndKey is greater than the EndKey of the requested range.

The above execution process has an obvious problem - every time we read data, we need to first access PD. This will put huge pressure on PD and affect request performance.

To solve this problem, tikv-client implemented a [RegionCache](https://github.com/pingcap/tidb/blob/v2.1.0-rc.1/store/tikv/region_cache.go#L50) component to cache Region information. When we need to locate the Region where a key is located, if RegionCache hits, we don't need to access PD. Inside RegionCache, there are two data structures saving Region information, one is a [map](https://github.com/pingcap/tidb/blob/v2.1.0-rc.1/store/tikv/region_cache.go#L55), another is a [b-tree](https://github.com/pingcap/tidb/blob/v2.1.0-rc.1/store/tikv/region_cache.go#L56). The map can quickly find Region by region ID, while b-tree can find the Region containing a key.

Strictly speaking, the Region information saved on PD is also a cache layer. The truly latest Region information is stored on tikv-server. Each tikv-server decides when to split Regions by itself, and reports the information to PD when Regions change. PD uses the reported Region information to satisfy tidb-server's query needs.

When we get Region information from cache and send requests, tikv-server will verify the Region information to ensure the request's Region information is correct.

If Region information has changed due to Region splits or Region migration, the requested Region information will be outdated, and tikv-server will return Region errors. When encountering Region errors, we need to [clear RegionCache](https://github.com/pingcap/tidb/blob/v2.1.0-rc.1/store/tikv/region_cache.go#L318), [get the latest Region information](https://github.com/pingcap/tidb/blob/v2.1.0-rc.1/store/tikv/region_cache.go#L329), and resend the request.

## How to establish and maintain connection with tikv-server

When TiDB locates the tikv-server where a key is located, it needs to establish connection with TiKV. We all know that establishing and closing TCP connections has considerable overhead and increases latency. Using a connection pool can save this overhead. TiDB and tikv-server maintain a connection pool [connArray](https://github.com/pingcap/tidb/blob/v2.1.0-rc.1/store/tikv/client.go#L83).

TiDB and TiKV communicate through gRPC, and gRPC supports multiplexing on a single TCP connection, so multiple concurrent requests can execute on a single connection without blocking each other.

Theoretically, a tidb-server only needs to maintain one connection with a tikv-server, but during performance testing, we found that a single connection becomes a performance bottleneck when concurrency is high. So in actual implementation, tidb-server maintains multiple connections for each tikv-server address, and [uses round-robin algorithm to select connections](https://github.com/pingcap/tidb/blob/v2.1.0-rc.1/store/tikv/client.go#L159) to send requests. The number of connections can be configured in the [config](https://github.com/pingcap/tidb/blob/v2.1.0-rc.1/config/config.toml.example#L215) file, default is 16.

## How to send RPC request

tikv-client implements the [kv.Storage](https://github.com/pingcap/tidb/blob/v2.1.0-rc.1/kv/kv.go#L247) interface through the [tikvStore](https://github.com/pingcap/tidb/blob/v2.1.0-rc.1/store/tikv/kv.go#L127) type. We can understand tikvStore as a wrapper of tikv-client. External calls to `kv.Storage` interface don't need to care about RPC details - RPC requests are initiated by tikvStore to implement the `kv.Storage` interface.

Implementing different `kv.Storage` interfaces requires sending different RPC requests. For example, implementing [Snapshot.BatchGet](https://github.com/pingcap/tidb/blob/v2.1.0-rc.1/kv/kv.go#L233) needs [tikvpb.TikvClient.KvBatchGet](https://github.com/pingcap/tidb/blob/v2.1.0-rc.1/vendor/github.com/pingcap/kvproto/pkg/tikvpb/tikvpb.pb.go#L61) method; implementing [Transaction.Commit](https://github.com/pingcap/tidb/blob/v2.1.0-rc.1/kv/kv.go#L128) needs multiple methods like [tikvpb.TikvClient.KvPrewrite](https://github.com/pingcap/tidb/blob/v2.1.0-rc.1/vendor/github.com/pingcap/kvproto/pkg/tikvpb/tikvpb.pb.go#L57), [tikvpb.TikvClient.KvCommit](https://github.com/pingcap/tidb/blob/v2.1.0-rc.1/vendor/github.com/pingcap/kvproto/pkg/tikvpb/tikvpb.pb.go#L58).

In tikvStore's implementation, it doesn't directly call RPC methods, but calls through a [Client](https://github.com/pingcap/tidb/blob/v2.1.0-rc.1/store/tikv/client.go#L76) interface. The main purpose of this abstraction layer is to allow different implementations at the lower layer. For example, [mocktikv implements the Client interface](https://github.com/pingcap/tidb/blob/v2.1.0-rc.1/store/mockstore/mocktikv/rpc.go#L493) for testing, implementing through local calls without needing real RPC calls.

[rpcClient](https://github.com/pingcap/tidb/blob/v2.1.0-rc.1/store/tikv/client.go#L180) is the Client implementation that actually makes RPC requests, sending RPC requests by calling [tikvrpc.CallRPC](https://github.com/pingcap/tidb/blob/v2.1.0-rc.1/store/tikv/tikvrpc/tikvrpc.go#L419). `tikvrpc.CallRPC` then calls [the code generated for each specific RPC](https://github.com/pingcap/tidb/blob/v2.1.0-rc.1/vendor/github.com/pingcap/kvproto/pkg/tikvpb/tikvpb.pb.go#L152). At the generated code layer, it's already at the gRPC framework layer, which we won't analyze further. Interested readers can research gRPC's implementation.

## How to handle various errors

We mentioned earlier that RPC requests are sent through the [Client](https://github.com/pingcap/tidb/blob/v2.1.0-rc.1/store/tikv/client.go#L76) interface, but actually this interface isn't directly called by tikvStore's various methods, but through a [RegionRequestSender](https://github.com/pingcap/tidb/blob/v2.1.0-rc.1/store/tikv/region_request.go#L46) object.

`RegionRequestSender`'s main work besides sending RPC requests is handling various retryable errors, like network errors and some Region errors.

**RPC request errors mainly fall into two categories: Region errors and network errors.**

[Region errors](https://github.com/pingcap/tidb/blob/v2.1.0-rc.1/store/tikv/tikvrpc/tikvrpc.go#L359) are returned by tikv-server in the response after receiving requests. Common ones include:

1. [NotLeader](https://github.com/pingcap/tidb/blob/v2.1.0-rc.1/vendor/github.com/pingcap/kvproto/pkg/errorpb/errorpb.pb.go#L207)
   This error usually occurs due to Region scheduling. For load balancing, PD may schedule a hot Region's leader to an idle tikv-server, but requests can only be handled by the leader. When encountering this error, tikv-client needs to retry by sending the request to the new leader.

2. [StaleEpoch](https://github.com/pingcap/tidb/blob/v2.1.0-rc.1/vendor/github.com/pingcap/kvproto/pkg/errorpb/errorpb.pb.go#L210)
   This error mainly occurs due to Region splits. When data volume in a Region increases, it splits into multiple new Regions. The new Regions cover different ranges, so executing directly could return incorrect results. Therefore TiKV rejects the request. tikv-client needs to get the latest Region information from PD and retry.

3. [ServerIsBusy](https://github.com/pingcap/tidb/blob/v2.1.0-rc.1/vendor/github.com/pingcap/kvproto/pkg/errorpb/errorpb.pb.go#L211)
   This error usually occurs because tikv-server has accumulated too many requests to process. If tikv-server doesn't reject the request, the queue will keep growing and the request may timeout before being processed. As a protection mechanism, tikv-server returns an error early, letting the client wait a while before retrying.

The other category is network errors, which are returned as errors from [SendRequest](https://github.com/pingcap/tidb/blob/v2.1.0-rc.1/store/tikv/region_request.go#L129). These errors usually mean the tikv-server didn't respond normally, possibly due to network isolation or tikv-server being down. When tikv-client encounters such errors, it calls the [OnSendFail](https://github.com/pingcap/tidb/blob/v2.1.0-rc.1/store/tikv/region_request.go#L140) method to handle them, [dropping all regions](https://github.com/pingcap/tidb/blob/v2.1.0-rc.1/store/tikv/region_cache.go#L453) on this failed tikv-server from RegionCache to avoid other requests encountering the same error.

When encountering retryable errors, we need to wait before retrying. We need to ensure the wait time between retries is neither too short nor too long - too short causes unnecessary requests increasing system pressure and overhead, too long increases request latency. We use exponential backoff algorithm to calculate wait time before each retry, implemented in [Backoffer](https://github.com/pingcap/tidb/blob/v2.1.0-rc.1/store/tikv/backoff.go#L176).

When executing a SQL statement at the upper level, at the tikv-client level, multiple sequential or concurrent requests are triggered and sent to multiple tikv-server. To ensure the timeout of the upper SQL statement, we need to consider not only a single RPC request but also a query's overall timeout.

To solve this problem, `Backoffer` implements a [fork](https://github.com/pingcap/tidb/blob/v2.1.0-rc.1/store/tikv/backoff.go#L267) function. When sending each sub-request, it needs to fork to make a `child Backoffer`. The `child Backoffer` is responsible for retries of a single RPC, recording the wait time from the `parent Backoffer` to ensure total wait time won't exceed query timeout.

For different errors, wait times differ. When creating each `Backoffer`, it will [create different backoff functions according to different types](https://github.com/pingcap/tidb/blob/v2.1.0-rc.1/store/tikv/backoff.go#L96).

**The above covers the first part about tikv-client. In the next article we'll introduce in detail the [copIterator](https://github.com/pingcap/tidb/blob/v2.1.0-rc.1/store/tikv/coprocessor.go#L354) for achieving distributed calculations and [twoPCCommiter](https://github.com/pingcap/tidb/blob/v2.1.0-rc.1/store/tikv/2pc.go#L66) for implementing distributed transactions.**

> Click to see more [TiDB source code reading series articles](https://cn.pingcap.com/blog/tag/tidb-source-code-reading/) 
