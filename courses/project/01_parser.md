# Project: SQL Parser Implementation

This project guides you through implementing a SQL parser for TinySQL, building upon the concepts covered in the source code reading 
([materials](../tidb-source-code-reading/5_parser.md)).

## Overview

The parser is a crucial component that transforms SQL text into an Abstract Syntax Tree (AST). This project will help you understand:
- How SQL statements are processed in TinySQL
- The role of lexical and syntactic analysis
- How to use Goyacc for parser generation
- How to implement specific SQL syntax parsing

## Prerequisites Reading

Before starting this project, make sure to read and understand:
1. [01_architecture.md](../tidb-source-code-reading/2_architecture.md) - System architecture
2. [02_processing_flow.md](../tidb-source-code-reading/3_processing_flow.md) - SQL processing flow
3. [03_parser.md](../tidb-source-code-reading/5_parser.md) - Parser implementation details

## Project Tasks

### Task 1: Understanding the Parser Framework

1. Study the existing parser components:
   - Lexical analyzer in `parser/lexer.go`
   - Token definitions in `parser/misc.go`
   - Grammar rules in `parser/parser.y`
   - AST node definitions in `ast/` package

2. Understand how different components work together:
   ```
   SQL Text → Lexer → Tokens → Parser → AST
   ```

### Task 2: Implementing JoinTable Syntax

Your main task is to implement the `JoinTable` syntax in the parser. This involves:

1. Adding grammar rules for JOIN operations:
   - INNER JOIN
   - LEFT JOIN
   - RIGHT JOIN
   - CROSS JOIN

2. Creating appropriate AST nodes for join operations

3. Implementing the necessary actions in parser.y

Example JOIN syntax to support:
```sql
SELECT * FROM t1 JOIN t2 ON t1.id = t2.id;
SELECT * FROM t1 LEFT JOIN t2 ON t1.id = t2.id;
SELECT * FROM t1 RIGHT JOIN t2 USING (id);
```

## Implementation Guide

1. Locate the JOIN-related sections in parser.y:
   ```yacc
   TableRef:
       TableFactor
       | JoinTable
       ;
   ```

2. Implement the JoinTable rules following MySQL syntax:
   ```yacc
   JoinTable:
       TableRef JoinType TableFactor JoinCondition
       ;
   ```

3. Add necessary token definitions and AST nodes

4. Test your implementation using the provided test cases

## Building and Testing

1. Generate the parser:
   ```bash
   cd parser
   make
   ```

2. Run the tests:
   ```bash
   cd ..
   make test-proj2
   ```

## Grading

Your implementation will be graded based on:
1. Passing the `TestDMLStmt` test suite
2. Correct handling of different JOIN types
3. Proper AST construction
4. Code quality and documentation

## Additional Resources

1. [MySQL JOIN Syntax Documentation](https://dev.mysql.com/doc/refman/8.0/en/join.html)
2. [Goyacc Documentation](https://github.com/cznic/goyacc)
3. [TiDB Parser Design](https://github.com/pingcap/tidb/blob/master/parser/README.md)

## Common Issues and Solutions

1. Parser conflicts:
   - Check for ambiguous grammar rules
   - Use precedence declarations if needed
   - Review the goyacc debug output

2. AST construction:
   - Ensure all necessary fields are populated
   - Verify parent-child relationships
   - Check for proper node types

3. Testing failures:
   - Compare AST output with expected structure
   - Check for missing edge cases
   - Verify error handling

## Next Steps

After completing this project, you'll be ready to move on to:
1. [Query Execution](../executor/README.md)
2. [Query Optimization](../optimizer/README.md)

Remember to reference the source code reading materials as you implement each feature.
