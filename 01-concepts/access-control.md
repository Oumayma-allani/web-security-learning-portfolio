# Access Control

## Simple definition

Access control is the set of rules that determines who or what is allowed to perform actions or access resources.

Authentication answers:

```text
Who are you?
```

Access control answers:

```text
What are you allowed to do?
```

## Access control models

### Programmatic access control

Access rules are checked by application code, often using permissions stored in a database or access matrix.

Example:

```text
The app checks whether a user can view, edit, or delete a resource.
```

### Discretionary Access Control — DAC

The owner of a resource decides who can access it.

Example:

```text
A file owner grants another user read or edit permission.
```

### Mandatory Access Control — MAC

Access is controlled centrally by strict rules. Users cannot freely change permissions.

Example:

```text
Military-style clearance levels such as confidential, secret, and top secret.
```

### Role-Based Access Control — RBAC

Permissions are assigned to roles, and users are assigned to roles.

Example:

```text
Admin, manager, support agent, normal user.
```

## Types of access control

### Vertical access control

Different roles have access to different functions.

Example:

```text
Admin can delete users.
Normal users cannot.
```

Broken vertical access control can lead to privilege escalation.

### Horizontal access control

Users with the same role should only access their own resources.

Example:

```text
User A can view their own basket.
User A must not view User B's basket.
```

Broken horizontal access control often appears as IDOR/BOLA.

### Context-dependent access control

An action is allowed only in the correct state or process step.

Example:

```text
A user should not modify a shopping cart after payment is completed.
```

### Location-based access control

Access is allowed or blocked depending on location.

Example:

```text
A streaming service restricts content by country.
```

Weakness:

```text
Location-based controls can sometimes be bypassed with VPNs, proxies, or weak client-side checks.
```

### Referer-based access control

The application checks the `Referer` header to decide whether a request is allowed.

Risk:

```text
The Referer header is not a strong security control and can be missing or manipulated.
```

## Common broken access control issues

### Vertical privilege escalation

A normal user gains access to admin functionality.

Examples:

- Accessing `/admin`
- Calling an admin-only API endpoint
- Changing role-related fields in a request

### Horizontal privilege escalation

A user accesses another user’s resources.

Example:

```text
/user/account?id=123
/user/account?id=124
```

If changing the ID gives access to another account, this is likely an IDOR/BOLA-style issue.

### Unprotected functionality

Some admin or sensitive functions may exist but not be linked in the UI.

Example:

```text
/robots.txt
Disallow: /admin/
Disallow: /backup/
```

Important rule:

```text
robots.txt is not security.
```

It only tells polite crawlers what not to crawl. It does not protect the paths.

### URL-based access control bypass

Some applications rely on front-end routes, reverse proxies, frameworks, or gateways to block access.

Risky headers include:

```text
X-Original-URL
X-Rewrite-URL
```

Example flow:

```text
Attacker sends request to: /
Header: X-Original-URL: /admin/deleteUser

Access control checks only /
Backend rewrites request to /admin/deleteUser
Admin function may execute
```

Root issue:

```text
The access control check and the backend route processing do not agree on the real target URL.
```


## Root cause

Access control vulnerabilities usually happen because:

- Authorization checks are missing.
- Checks exist only in the UI.
- The server trusts user-controlled IDs.
- The app does not verify resource ownership.
- Admin endpoints are hidden but not protected.
- Different layers apply inconsistent access rules.
- The app relies on weak signals such as headers or location.

## Defensive lessons

- Deny access by default.
- Enforce authorization server-side.
- Never rely on hidden URLs or UI controls.
- Verify ownership for every object access.
- Use a central access control mechanism.
- Make developers explicitly declare required permissions.
- Log and monitor access control failures.
- Test access control thoroughly with different users and roles.

## Beginner mistakes to avoid

- Confusing authentication with authorization.
- Testing only the UI and not the API.
- Assuming hidden endpoints are protected.
- Forgetting to test with two users.
- Looking only at status codes and not checking response data.
- Thinking that `robots.txt` protects sensitive paths.

## Related Topics

- IDOR vs BOLA terminology.
- Access control testing in APIs.
- Role-based vs attribute-based access control.
- Method-based bypasses such as GET vs POST.
- Path normalization and URL parsing issues.
