# SQL Data Guard Plugin for Dify

This is a Dify Python plugin that integrates [sql-data-guard](https://github.com/thalesgroup/sql-data-guard) to validate SQL queries and enforce security policies.

## Overview

The SQL Data Guard plugin allows Dify users to enforce table/column restrictions, detect malicious payloads, and modify non-compliant queries dynamically. This is particularly useful when using LLMs to generate SQL queries, ensuring that only permitted data is accessed and SQL injection attacks are prevented.

For more information about the main project, visit the [sql-data-guard repository](https://github.com/thalesgroup/sql-data-guard).

## Plugin Inputs

- `sql` (string, required) – SQL query to validate  
- `config` (string, required) – Policy configuration; can be static or templated  
- `dialect` (string, optional) – SQL dialect for parsing

## Example

Config Input:
```json
{
  "tables": [
    {
      "table_name": "orders",
      "columns": ["id", "product_name", "account_id"],
      "restrictions": [{"column": "account_id", "value": 123}]
    }
  ]
}
```
SQL Input:
```sql
SELECT id, name FROM orders WHERE 1 = 1
```

Result Output Variables:

| Field         | Description                                                                                     | Example Value                                                                                                                                                                                            |
|---------------|---------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| allowed       | Boolean indicating whether the query complies with the policy configuration                     | false                                                                                                                                                                                                     |
| errors        | List of validation errors found in the query (e.g., restricted columns, injection attempts)     | Column name not allowed. Column removed from SELECT clause;<br>Always-True expression is not allowed;<br>Missing restriction for table: orders column: account_id value: 123                                |
| fixed         | Modified query that complies with the policy (only present if query was non-compliant)           | SELECT id, product_name, account_id FROM orders WHERE account_id = 123                                                                                                                                   |
| verified_sql  | The validated/fixed SQL query ready for execution                                                | SELECT id, product_name, account_id FROM orders WHERE account_id = 123                                                                                                                                   |
| risk          | Risk score between 0 and 1 indicating the severity of policy violations (0 = safe, 1 = unsafe) | 0.7                                                                                                                                                                                                       |

## Example Workflow

The following diagrams illustrate how SQL Data Guard protects your application when using LLMs to generate SQL queries:

### Without SQL Data Guard (Unsafe)
```
User Question
     ↓
   LLM
     ↓
Database ← (Unvalidated SQL)
     ↓
  Output
```
**Risk**: LLM-generated SQL could access restricted data or exploit injection vulnerabilities. The query runs directly on the database without any security validation, making it vulnerable to:
- Unauthorized data access
- SQL injection attacks
- Privilege escalation
- Accidental data exposure

### With SQL Data Guard (Secure)
```
User Question
     ↓
   LLM
     ↓
SQL Data Guard (Validates against Policy)
     ↓
  Policy Compliant?
   /        \
 YES        NO
  |          |
  ↓          ↓
 DB        Blocked
  |          |
  ↓          ↓
Output     Error Response
```
**Security**: The plugin validates every query before execution:
- **If compliant**: The validated query proceeds to the database for execution
- **If non-compliant**: The query is blocked and returns an error response, preventing unauthorized access
- **Query modification**: Non-compliant queries can be automatically fixed to comply with your policy (e.g., removing restricted columns, adding missing restrictions)

This ensures that only permitted data is accessed and SQL injection attacks are prevented.
