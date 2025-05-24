# TinySQL

TinySQL is a comprehensive course designed to teach you how to implement a distributed relational database in Go. The name TinySQL indicates it is a simplified version of [TiDB](https://github.com/pingcap/tidb).

## Prerequisites

- Experience with Go is required. If not, it is recommended to learn [A Tour of Go](https://tour.golang.org/) first.
- Basic understanding of database concepts and SQL
- Familiarity with distributed systems concepts (recommended)

## Course Structure

The course combines theoretical understanding through source code reading with hands-on implementation projects. Each module includes both learning materials and practical projects.

### Module 1: Architecture and Fundamentals
- **Architecture Overview** [Reading](./courses/tidb-source-code-reading/2_architecture.md)
- **Processing Flow** [Reading](./courses/tidb-source-code-reading/3_processing_flow.md)
- **SQL and Parser**
  - Parser Implementation [Reading](./courses/tidb-source-code-reading/5_parser.md)
  - Project: SQL Parser [Implementation](./courses/project/parser/parser.md)
  - Project: SQL Foundations [Implementation](./courses/proj-fundamentals/README.md)

### Module 2: Query Processing and Optimization
- **Query Execution**
  - Select Statement Processing [Reading](./courses/tidb-source-code-reading/6_select.md)
  - Project: Query Executor [Implementation](./courses/proj-executor/README.md)
- **Query Optimization**
  - Logical Optimization [Reading](./courses/tidb-source-code-reading/7_logical_optimization.md)
  - Cost-based Optimization [Reading](./courses/tidb-source-code-reading/8_cost_optimization.md)
  - Advanced Optimization [Reading](./courses/tidb-source-code-reading/21_logical_optimize_p2.md)
  - Project: Query Optimizer [Implementation](./courses/proj-optimizer/README.md)
- **Join Operations**
  - Hash Join Implementation [Reading](./courses/tidb-source-code-reading/9_hash_join.md)
  - Sort Merge Join [Reading](./courses/tidb-source-code-reading/15_sort_merge_join.md)

### Module 3: Data Storage and Access
- **Storage Structures**
  - Chunk Data Structure [Reading](./courses/tidb-source-code-reading/10_chunk.md)
  - Index Implementation [Reading](./courses/tidb-source-code-reading/11_index.md)
- **Statistics and Optimization**
  - Statistics Collection [Reading](./courses/tidb-source-code-reading/12_statistical.md)
  - Statistics Implementation [Reading](./courses/tidb-source-code-reading/14_statistical_impl.md)
  - Range Calculation [Reading](./courses/tidb-source-code-reading/13_range_calculation.md)
- **Data Organization**
  - Table Partitioning [Reading](./courses/tidb-source-code-reading/20_tbl_partition.md)
  - Project: Table Storage [Implementation](./courses/proj-storage/README.md)

### Module 4: Advanced Features
- **DDL Operations**
  - DDL Implementation [Reading](./courses/tidb-source-code-reading/17_ddl.md)
  - Project: DDL Implementation [Implementation](./courses/proj-ddl/README.md)
- **Transaction Processing**
  - Project: Percolator Transaction [Implementation](./courses/proj-transaction/README.md)
- **TiKV Integration**
  - TiKV Implementation Part 1 [Reading](./courses/tidb-source-code-reading/18_tikv_impl_p1.md)
  - TiKV Implementation Part 2 [Reading](./courses/tidb-source-code-reading/19_tikv_impl_p2.md)
- **Additional Features**
  - Insert Operations [Reading](./courses/tidb-source-code-reading/4_insert.md)
  - Insert Types [Reading](./courses/tidb-source-code-reading/16_insert_type.md)
  - Hash Aggregation [Reading](./courses/tidb-source-code-reading/22_hash_aggregation.md)
  - Prepared Statements [Reading](./courses/tidb-source-code-reading/23_prepare_statement.md)
  - Binlog [Reading](./courses/tidb-source-code-reading/24_binlog.md)

## System Architecture

TinySQL follows a distributed SQL architecture similar to TiDB:

- Protocol Layer: MySQL protocol support
- SQL Layer: Parser, Optimizer, Executor
- KV Layer: Distributed storage interface

## Deployment

### Build

```bash
make
```

### Run & Play

1. **Standalone Mode**:
   ```bash
   ./bin/tidb-server
   mysql -h127.0.0.1 -P4000 -uroot
   ```

2. **Cluster Mode** (with TinyKV):
   ```bash
   mkdir -p data
   ./tinyscheduler-server
   ./tinykv-server -path=data
   ./tidb-server --store=tikv --path="127.0.0.1:2379"
   ```

## Related Projects

- [TinyKV](https://github.com/pingcap-incubator/tinykv) - The distributed KV storage implementation course

## Contributing

Contributions are welcome! Please read our contributing guidelines before submitting pull requests.

## License

TinySQL is under the Apache 2.0 license. See the [LICENSE](./LICENSE) file for details.
