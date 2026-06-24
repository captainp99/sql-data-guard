### **Column Masking / Redaction**

Instead of *removing* a sensitive column from the `SELECT` clause (which changes the
result shape and can break downstream applications), sql-data-guard can rewrite the
column into a **masking expression**. The query keeps returning a column with the same
output name, but the sensitive value never leaves the trust boundary.

Masking is declared per table via a `column_masks` list. Every masked column must also
appear in that table's `columns` allow-list.

```json
{
  "table_name": "users",
  "columns": ["id", "email", "credit_card", "ssn"],
  "column_masks": [
    { "column": "credit_card", "policy": "partial", "show_last": 4 },
    { "column": "email", "policy": "hash" },
    { "column": "ssn", "policy": "redact" }
  ]
}
```

**Supported policies:**

| Policy | Behavior | Options |
|---|---|---|
| `redact` | Replaces the value with a constant string (default `'****'`). | `replacement` (string) |
| `hash` | Replaces the value with a one-way hash (`MD5(col)`), portable across dialects. | — |
| `partial` | Keeps the last `show_last` characters and masks the rest (e.g. `****1234`). | `show_last` (int, default `4`), `replacement` (string) |

When a query selects a masked column, the result is reported as **not allowed** (the
original query would have exposed raw sensitive data) and the compliant, masked query
is returned in the `fixed` field — run that instead. Example:

```
SELECT id, credit_card FROM users
  -> fixed: SELECT id, '****' || SUBSTRING(CAST(credit_card AS VARCHAR), -4) AS credit_card FROM users
```

The masking expression is generated per `dialect`, so it renders to valid,
executable SQL for the target database.

### **Restriction Schema and Validation**

Restrictions are utilized to validate queries by ensuring that only supported operations are applied to the columns of tables.
The restrictions determine how values are compared against table columns in SQL queries. Below is a breakdown of how the restrictions are validated, the available operations, and the conditions under which they are applied.

#### **Supported Operations**

The following operations are supported in the restriction schema:

- **`=`**: Equal to – Checks if the column value is equal to a given value.
- **`>`**: Greater than – Checks if the column value is greater than a specified value.
- **`<`**: Less than – Checks if the column value is less than a given value.
- **`>=`**: Greater than or equal to – Checks if the column value is greater than or equal to a specified value.
- **`<=`**: Less than or equal to – Checks if the column value is less than or equal to a given value.
- **`BETWEEN`**: Between two values – Validates if the column value is within a specified range.
- **`IN`**: In a specified list of values – Validates if the column value matches any of the values in the given list.

#### **Restriction Structure**
Each restriction in the configuration consists of the following keys:
- **`column`**: The name of the column the restriction is applied to (e.g., `"price"` or `"order_id"`).
- **`value`** or **`values`**: The value(s) to compare the column against:
  - If the operation is `BETWEEN`, the `values` field should contain a list of two numeric values, representing the lower and upper bounds of the range.
  - For operations like `IN` or comparison operations (e.g., `=`, `>`, `<=`), the `value` or `values` field will contain one or more values to compare.
- **`operation`**: The operation to apply to the column. This could be any of the supported operations, such as `BETWEEN`, `IN`, `=`, `>`, etc.

#### **Validation Rules for Specific Operations**

1. **BETWEEN**:
   - The `BETWEEN` operation requires the `values` field to contain a list of exactly two numeric values. The first value must be less than the second.
   - **Example**: 
     ```
     "operation" : "BETWEEN", 
     "values": [100, 200]
     ```
   - In this case, the `price` column must have a value between 100 and 200.

2. **IN**:
   - The `IN` operation requires the `values` field to be a list containing multiple values to match the column against. The values can be of types such as integers, floats, or strings.
   - **Example**:
     ```
     "operation": "IN", 
     "values": [100, 200, 300]
     ```
   - In this case, the `category` column will be checked to see if its value matches one of the values in the list: 100, 200, or 300.

3. **Comparison Operations (>=, <=, =, <, >)**:
   - These operations apply a comparison between the column and a single value. The value must be numeric for comparison operations like `>=`, `<`, etc.
   - **Example**: 
        ```
     "operation": ">=", 
     "value": 100
     ```
   - In this case, the `price` column must have a value greater than or equal to 100.

#### **Error Handling and Restrictions**

The validation function checks that the restrictions adhere to the following rules and raises errors if any of these conditions are violated:

1. **Unsupported Operations**:
   - If an unsupported operation is used in the configuration, an `UnsupportedRestrictionError` is raised. Only operations listed in the "Supported Operations" section are allowed.

2. **Missing Columns or Tables**:
   - If a table in the configuration is missing either the `columns` or `table_name` fields, or if no tables are provided in the configuration, a `ValueError` is raised. Every table must specify these fields.

3. **Invalid Data Types**:
   - If the `value` or `values` in the restriction do not match the expected data types (e.g., using non-numeric values for comparison operations), a `ValueError` will be raised.
 For example:
   - A `BETWEEN` operation that doesn’t provide a list of two numeric values will trigger an error:
     ```
     "operation": "BETWEEN", 
     "values": ["A", "B"]
     ```
     This would raise an error because the values are not numeric.

4. **Invalid `IN` Format**:
   - If the `IN` operation is provided with invalid data types (e.g., a list with mixed types like numbers and strings), it will also result in a validation error:
     ```
     "operation": "IN", 
     "values": [100, "Electronics"]
     ```
     This would raise an error because the values are not consistently of the same data type.
