# Project: DDL (Data Definition Language) Implementation

This project guides you through implementing key DDL operations in TinySQL, building upon the concepts covered in the source code reading materials ([18_ddl_implementation.md](../../tidb-source-code-reading/18_ddl_implementation.md)).

## Overview

The DDL module is responsible for handling schema changes in TinySQL. This project will help you understand:
- How asynchronous schema changes work
- The implementation of DDL operations (CREATE, ALTER, DROP)
- How to maintain consistency during schema changes
- The F1 schema change algorithm

## Prerequisites Reading

Before starting this project, make sure to read and understand:
1. [18_ddl_implementation.md](../../tidb-source-code-reading/18_ddl_implementation.md) - DDL system design
2. [19_ddl_insert_type.md](../../tidb-source-code-reading/19_ddl_insert_type.md) - Insert handling
3. [Online, Asynchronous Schema Change in F1](http://static.googleusercontent.com/media/research.google.com/zh-CN//pubs/archive/41376.pdf) - Core algorithm

## Architecture Overview

The DDL system consists of several key components:

1. DDL Manager (`ddl.go`):
   - Manages DDL jobs and their lifecycle
   - Coordinates schema changes across the cluster

2. DDL API (`ddl_api.go`):
   - Provides high-level DDL operations
   - Handles job creation and queuing
   - Manages job state transitions

3. DDL Worker (`ddl_worker.go`):
   - Executes DDL jobs asynchronously
   - Implements the F1 schema change algorithm
   - Manages job history

4. Schema Syncer (`syncer.go`):
   - Synchronizes schema versions across nodes
   - Handles version conflicts and resolution

## Project Tasks

### Task 1: Schema Version Management
Implement `updateVersionAndTableInfo` in `ddl/table.go`:
```go
// updateVersionAndTableInfo updates the schema version and table info.
func updateVersionAndTableInfo(
    t *meta.Meta, job *model.Job,
    tblInfo *model.TableInfo,
    newVersion int64) error {
    // Your implementation here
}
```

Key requirements:
- Update the schema version atomically
- Store the new table information
- Handle error conditions properly

### Task 2: Add Column Implementation
Implement `onAddColumn` in `ddl/column.go`:
```go
func onAddColumn(d *ddl, t *meta.Meta, job *model.Job) (ver int64, err error) {
    // Your implementation here
}
```

Requirements:
- Handle column addition states
- Validate column constraints
- Update table metadata
- Manage schema version changes

### Task 3: Drop Column Implementation
Implement `onDropColumn` in `ddl/column.go`:
```go
func onDropColumn(d *ddl, t *meta.Meta, job *model.Job) (ver int64, err error) {
    // Your implementation here
}
```

Requirements:
- Handle column removal states
- Validate dependencies
- Clean up column data
- Update table metadata

## Implementation Guide

### Schema Version Management
1. Read the current schema version
2. Create a new version number
3. Update table information atomically
4. Handle transaction rollback

Example workflow:
```go
// 1. Get current version
currentVersion := getSchemaVersion()

// 2. Create new version
newVersion := currentVersion + 1

// 3. Update atomically
err := t.UpdateTableInfo(schemaID, tblInfo, newVersion)
```

### Column Operations
1. Implement state machine transitions
2. Handle data backfilling
3. Manage schema versions
4. Validate constraints

Example state machine:
```go
switch job.SchemaState {
case model.StateNone:
    // Initialize
case model.StateDeleteOnly:
    // Handle delete-only state
case model.StateWriteOnly:
    // Handle write-only state
case model.StateWriteReorg:
    // Handle write reorganization
case model.StatePublic:
    // Finalize changes
}
```

## Building and Testing

1. Run specific tests:
   ```bash
   go test -v -check.f TestAddColumn
   go test -v -check.f TestDropColumn
   go test -v -check.f TestColumnChange
   ```

2. Run all project tests:
   ```bash
   make test-proj3
   ```

## Common Issues and Solutions

1. Schema Version Conflicts:
   - Always check current version before updates
   - Use atomic operations for version changes
   - Handle rollback scenarios

2. Data Consistency:
   - Follow the F1 algorithm states strictly
   - Validate data during state transitions
   - Handle partial failures

3. Performance Considerations:
   - Minimize blocking operations
   - Use appropriate batch sizes
   - Consider impact on running transactions

## Grading

Your implementation will be graded based on:
1. Passing all test cases in `make test-proj3`
2. Proper handling of edge cases
3. Code quality and documentation
4. Performance considerations

## Next Steps

After completing this project, you'll be ready to move on to:
1. [Query Execution](../executor/README.md)
2. [Transaction Processing](../transaction/README.md)

## Additional Resources

1. [TiDB DDL Architecture](https://docs.pingcap.com/tidb/stable/tidb-ddl)
2. [Online Schema Changes](https://www.percona.com/blog/2018/08/29/online-schema-changes/)
3. [F1: A Distributed SQL Database](https://research.google/pubs/pub41344/)
