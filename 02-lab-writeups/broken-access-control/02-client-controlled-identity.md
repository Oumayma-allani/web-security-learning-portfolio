# Client-Controlled Identity: Forged Feedback & Forged Review

## Category

Broken Access Control / Client-Controlled Identity

## Platform

OWASP Juice Shop

## Covered Challenges

* Forged Feedback
* Forged Review

## Objective

Understand how trusting identity fields sent by the client can allow a user to perform actions as another user.

## Core Idea

In both challenges, the backend accepted identity-related values directly from the request body.

This is dangerous because the authenticated user should be determined server-side from the JWT/session, not from fields that the client can modify.

## Challenge 1: Forged Feedback

### Endpoint

```http
POST /api/Feedbacks/
```

### Normal request body

```json
{
  "UserId": 24,
  "captchaId": 2,
  "captcha": "34",
  "comment": "test feedback",
  "rating": 2
}
```

### What I observed

The request body contained a `UserId` field.

This was suspicious because the backend already knows the authenticated user from the JWT/session.

The client should not be allowed to decide which user is submitting the feedback.

### Test performed

I changed the `UserId` value to another valid user ID:

```json
{
  "UserId": 4,
  "captchaId": 2,
  "captcha": "34",
  "comment": "test feedback",
  "rating": 2
}
```

### Result

The request succeeded.

This means the feedback was submitted using a user ID controlled by the client.

### Why it worked

The backend trusted the `UserId` value from the JSON request body instead of deriving the user identity from the authenticated session.

---

## Challenge 2: Forged Review

### Endpoint

```http
PUT /rest/products/{productId}/reviews
```

### Normal request body

```json
{
  "message": "test review",
  "author": "user@example.com"
}
```

### What I observed

The request body contained an `author` field.

This was suspicious because the review author should be determined by the authenticated user, not by a value sent from the frontend.

### Test performed

I changed the `author` value to another valid user email:

```json
{
  "message": "test review",
  "author": "victim@example.com"
}
```

### Result

The request succeeded.

The review was submitted as another user.

### Why it worked

The backend trusted the `author` field from the request body instead of using the authenticated user's email from the JWT/session.

---

## Root Cause

The root cause is broken access control caused by trusting client-controlled identity fields.

Main issues:

* The backend accepted user identity from request body fields.
* The server did not enforce that `UserId` or `author` matched the authenticated user.
* Identity was controlled by the client instead of being derived server-side.
* User IDs and emails could be discovered or reused from other parts of the application.

## Impact in a Real Application

An attacker could:

* Submit feedback as another user.
* Post reviews as another user.
* Forge user-generated content.
* Damage another user's reputation.
* Create misleading audit records.
* Abuse workflows that rely on trusted user identity.

This type of issue can affect reviews, comments, tickets, feedback forms, orders, account actions, and admin workflows.

## Remediation

Recommended fixes:

* Do not accept `UserId`, `author`, `email`, or similar identity fields from the client for authenticated actions.
* Derive the user identity server-side from the JWT/session.
* Enforce that the resource owner matches the authenticated user.
* Validate authorization on every state-changing API request.
* Reject requests where client-supplied identity differs from the authenticated identity.
* Add tests for IDOR and user identity spoofing.
* Log suspicious attempts to submit actions as another user.

## What I Learned

Broken Access Control is not always about changing an ID in the URL.

Sometimes the vulnerable value is inside the JSON body.

Important fields to inspect include:

```text
UserId
author
email
role
accountId
BasketId
OrderId
owner
```

The key question is:

```text
Should the client be allowed to control this value?
```

If the answer is no, the backend must ignore the client value and use the authenticated session instead.

## Memory Rule

```text
If a request body contains UserId, author, email, role, accountId, BasketId, or OrderId,
test whether the backend really verifies ownership.
```

For these challenges:

```text
Forged Feedback = change UserId.
Forged Review = change author email.
Same bug = backend trusts client-controlled identity.
Same fix = derive identity server-side from the authenticated session.
```

## Related Patterns

* Broken Access Control
* IDOR
* Client-controlled identity
* User identity spoofing
* Missing server-side authorization
* API authorization bypass
