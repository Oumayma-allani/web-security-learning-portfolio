# Authentication

## Simple definition

Authentication is the process of verifying that a user is who they claim to be.

Authentication answers:

```text
Who are you?
```

Authorization / access control answers:

```text
What are you allowed to access?
```

## Main authentication factors

### Knowledge factor

Something the user knows.

Examples:

- Password
- PIN
- Security question answer

### Possession factor

Something the user has.

Examples:

- Mobile phone
- Hardware security key
- Authenticator app
- Security token

### Inherence factor

Something the user is or does.

Examples:

- Fingerprint
- Face recognition
- Behavioral patterns

## How authentication vulnerabilities happen

Authentication vulnerabilities usually happen in two main ways:

### 1. Weak protection against brute force

The application allows attackers to try many credentials without enough rate limiting, lockout, or detection.

### 2. Broken authentication logic

A coding or business logic mistake lets attackers bypass login or abuse account-related features.

## Vulnerable areas

Authentication issues can appear in:

- Password-based login
- Multi-factor authentication
- Remember-me functionality
- Password reset
- Password change
- Account recovery
- Third-party login / OAuth

## Password-based login vulnerabilities

### Brute-forcing usernames

Usernames are often predictable.

Examples:

```text
admin
administrator
firstname.lastname
email addresses
```

Attackers may discover usernames from:

- Public profiles
- Error messages
- API responses
- HTML source
- Emails
- Different login behavior

### Username enumeration

Username enumeration happens when the application leaks whether a username exists.

Examples:

```text
Invalid user -> User does not exist
Valid user + wrong password -> Incorrect password
```

Other signs:

- Different status codes
- Different response lengths
- Different redirects
- Different response times
- Account lockout messages

### Brute-forcing passwords

Attackers try many passwords against one account.

Common defenses:

- Rate limiting
- Account lockout
- IP-based throttling
- CAPTCHA
- Monitoring and alerts
- Multi-factor authentication

### Password spraying

Password spraying means trying a few common passwords against many accounts.

Memory rule:

```text
Brute force = many passwords against one account
Password spraying = few passwords against many accounts
Credential stuffing = leaked credentials reused on another site
```

### Credential stuffing

Credential stuffing uses username/password pairs leaked from other breaches.

Risk:

```text
Many users reuse passwords across websites.
```

Account lockout may not be enough because each account may only be tried once or a few times.

## Flawed brute-force protection

### Failed counter reset

Some apps reset the failed-attempt counter after a successful login.

Risk:

```text
An attacker can fail several times, log in to their own account, then continue guessing.
```

### Account lockout leaks usernames

If the application says:

```text
This account is locked
```

it may reveal that the username exists.

### Weak IP rate limiting

If protection only blocks one IP address, attackers may try to appear from different IPs or abuse headers such as:

```text
X-Forwarded-For
```

Important note:

```text
Security controls should not blindly trust client-supplied IP headers.
```

## HTTP Basic Authentication

HTTP Basic Authentication sends credentials in the `Authorization` header.

Example format:

```text
Authorization: Basic base64(username:password)
```

Important rule:

```text
Base64 is encoding, not encryption.
```

Why it can be risky:

- Credentials are sent with every request.
- Weak HTTPS/HSTS exposes credentials.
- Brute-force protection is often weak.
- It has no built-in CSRF protection.
- Users may reuse the same credentials elsewhere.

## Multi-factor authentication vulnerabilities

### Strong vs weak 2FA

Good 2FA uses different factors.

Examples:

```text
Password + authenticator app
Password + hardware token
```

Weaker cases:

```text
Password + email code
Password + SMS code
```

Why SMS can be weaker:

- SIM swapping
- SMS interception
- Phone number takeover

### 2FA bypass by skipping the second step

Bad flow:

```text
Step 1: password accepted
Step 2: 2FA page shown
Problem: application already treats the user as logged in
```

Testing idea in labs:

```text
After password step but before 2FA, try accessing:
/my-account
/account
/profile
/admin
```

### Flawed two-factor verification logic

Weak design:

```text
Step 1: attacker logs in with own account
Step 2: app stores account identity in a weak cookie
Step 3: attacker changes cookie to victim account
Step 4: attacker submits/brute-forces victim’s 2FA code
```

Root issue:

```text
The app does not securely bind the 2FA step to the original authenticated session.
```

### Brute-forcing 2FA codes

2FA codes are often short:

```text
4 digits = 0000 to 9999
6 digits = 000000 to 999999
```

Defensive requirement:

```text
The application must rate-limit and block repeated 2FA attempts.
```

## Other authentication mechanisms

### Remember-me cookies

Remember-me cookies act like persistent login tokens.

Bad examples:

```text
cookie = username + timestamp
cookie = Base64(username:password)
cookie = weak hash without salt
```

Safer idea:

```text
Use a long, random, server-side validated token.
```

Important rule:

```text
A remember-me cookie is equivalent to a login token.
```

### Password reset

Bad practices:

- Sending the current password by email.
- Sending long-lived new passwords by email.
- Using predictable reset links.

Bad reset link:

```text
/reset-password?user=carlos
```

Better reset link:

```text
/reset-password?token=random-long-secret
```

The token should be:

- Random
- Long
- Single-use
- Short-lived
- Bound to the correct user

### Password change

Risky signs:

- Username in hidden field.
- No current password required.
- User identity can be changed in the request.
- Same weak logic as login page.

## OAuth / third-party authentication

OAuth is used when a website lets users log in using another provider.

Examples:

```text
Log in with Google
Log in with Facebook
Log in with GitHub
```

Important note:

```text
OAuth was originally designed for authorization, not authentication.
```

### Main parties

```text
Client application = website the user wants to log in to
Resource owner = the user
OAuth provider = Google / Facebook / GitHub
```

### Important parameters

```text
client_id      = identifies the client application
redirect_uri   = where the user is redirected after login
response_type  = OAuth flow type
scope          = requested data/permissions
state          = CSRF protection value
```

### Why OAuth can be vulnerable

OAuth is flexible and can be misconfigured.

Common problems:

- Weak `redirect_uri` validation.
- Missing `state` parameter.
- Access token leaked in the browser.
- Client trusts user data without verifying the token.
- Account linking flaws.

Memory rule:

```text
OAuth login = “Log in using another website”.
OAuth bugs often happen because the client app trusts redirects, tokens, or user data too much.
```

## Root cause

Authentication vulnerabilities usually happen because:

- Login logic is inconsistent.
- Error messages reveal too much.
- Rate limiting is weak.
- Tokens are predictable or long-lived.
- The app trusts client-side data.
- Password reset/change flows are not protected.
- 2FA is not enforced consistently.
- OAuth is misconfigured.

## Defensive lessons

- Use generic error messages.
- Apply rate limiting and monitoring.
- Use secure password hashing.
- Enforce MFA consistently.
- Protect password reset with strong tokens.
- Do not expose sensitive account state.
- Bind authentication steps to the server-side session.
- Never trust client-controlled identity fields.
- Validate OAuth parameters strictly.

## Beginner mistakes to avoid

- Confusing authentication with access control.
- Testing only the login form and ignoring reset/change flows.
- Forgetting username enumeration through response time or status code.
- Thinking Base64 is encryption.
- Assuming 2FA is safe just because it exists.
- Ignoring remember-me cookies.

## Related Topics

- OAuth flows in more detail.
- Session management.
- Secure cookie attributes.
- Password hashing basics.
- MFA bypass patterns.
