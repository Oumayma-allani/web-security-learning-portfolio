# SQL Injection

## Simple definition

SQL injection happens when user-controlled input is inserted into a database query without safe handling.

The core problem is that the application mixes SQL code and user data.

```text
Bad pattern:
SQL query string + user input

Safe pattern:
Fixed SQL query + user values passed separately
```

## Where it appears

SQL injection can appear in many entry points, for example:

- Login forms
- Search bars
- Product filters
- Product IDs
- URL parameters
- API body parameters
- Cookies
- Headers
- Stored data reused later in queries

## How to detect SQL injection

A systematic approach is to test each input point and look for meaningful differences in the response.

Common detection ideas:

- Submit a single quote `'` and look for errors or unusual behavior.
- Submit SQL syntax that should preserve the original value.
- Submit boolean conditions and compare responses.
- Test for time delays when results are not directly visible.
- Use out-of-band testing when supported in labs.

## Common examples

### 1. Retrieving hidden data

Example logic:

```sql
SELECT * FROM products WHERE category = 'Gifts' AND released = 1
```

If the application directly inserts the category value, an attacker may alter the query logic and retrieve products that should not be visible.

Root issue:

```text
User input changes the meaning of the WHERE clause.
```

### 2. Subverting application logic

Example login logic:

```sql
SELECT * FROM users WHERE username = 'wiener' AND password = 'blue'
```

If the username is inserted unsafely, an attacker may comment out the password condition and bypass authentication.

Security lesson:

```text
Authentication must never depend on unsafe string-built SQL queries.
```

### 3. UNION attacks

A UNION attack uses the SQL `UNION` operator to append results from another query.

Example idea:

```sql
' UNION SELECT username, password FROM users--
```

Goal:

- Retrieve data from another table.
- Match the number and type of columns expected by the original query.

What I need to remember:

- The original query and injected query must usually return the same number of columns.
- The selected column types must be compatible.
- UNION attacks are useful when query results are displayed in the response.

### 4. Blind SQL injection

Blind SQL injection happens when the data is not directly returned in the response.

Instead of reading data directly, the attacker infers information using side effects.

Common techniques:

#### Response differences

Inject true/false conditions and compare page content.

Example idea:

```text
Condition true  -> normal page or “Welcome back”
Condition false -> different page or missing content
```

#### Conditional errors

Trigger a database error only when a condition is true.

```text
Error response = condition true
Normal response = condition false
```

#### Time delays

Make the database wait only if a condition is true.

```text
Slow response = condition true
Fast response = condition false
```

#### Out-of-band interactions

Make the server/database contact an external domain controlled by the tester.

This is useful when no visible response difference exists.

## Second-order SQL injection

First-order SQL injection:

```text
HTTP request -> SQL query affected immediately -> impact happens now
```

Second-order SQL injection:

```text
HTTP request -> malicious input stored in database -> no immediate impact
Later request -> app reads stored value -> app uses it unsafely in SQL -> impact happens later
```

Important lesson:

```text
Stored data can become dangerous later if reused unsafely.
```

## Root cause

SQL injection usually happens because:

- User input is concatenated into SQL strings.
- Stored data is trusted too much.
- Developers assume some inputs are safe.
- Dynamic query parts are built without strict validation.
- The database interprets user data as SQL code.

## Main protection

Use parameterized queries / prepared statements.

### Vulnerable pattern

```java
String query = "SELECT * FROM products WHERE category = '" + input + "'";
```

Problem:

```text
User input becomes part of the SQL query structure.
```

### Safer pattern

```java
PreparedStatement statement = connection.prepareStatement(
  "SELECT * FROM products WHERE category = ?"
);
statement.setString(1, input);
```

The `?` is a placeholder. The SQL structure stays fixed, and the input is treated as data.

## Where parameterized queries work well

- `WHERE username = ?`
- `INSERT INTO users VALUES (?, ?)`
- `UPDATE users SET email = ? WHERE id = ?`

## Where parameterized queries do not work directly

Parameterized values usually cannot replace:

- Table names
- Column names
- `ORDER BY` fields
- `ASC` / `DESC`

For these cases, use a strict whitelist of allowed values.

## Defensive lessons

- Use parameterized queries.
- Avoid SQL string concatenation.
- Validate dynamic query parts with allowlists.
- Apply least privilege to database users.
- Do not expose detailed SQL errors to users.
- Treat stored data as untrusted.
- Test authentication, search, filters, and API parameters.

## Beginner mistakes to avoid

- Focusing only on payloads instead of understanding query logic.
- Confusing authentication bypass with access control issues.
- Forgetting that SQL injection can happen in cookies, headers, or stored values.
- Thinking that a WAF is a real fix.
- Assuming internal data is safe.

## Related Topics

- UNION attacks in depth.
- Blind SQL injection.
- Time-based SQL injection.
- Database-specific syntax.
- Second-order SQL injection examples.
