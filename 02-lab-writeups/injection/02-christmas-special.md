# Challenge: Christmas Special

## Category

Injection / SQL Injection

## Platform

OWASP Juice Shop

## Objective

Order the Christmas special offer of 2014.

## What I observed

The application exposes a product search API endpoint:

```http
GET /rest/products/search?q=
```

When I inspected the response in Burp Repeater, I noticed that products contain a field named:

```text
deletedAt
```

Visible products had:

```json
"deletedAt": null
```

This suggested that the application uses soft deletion.

A soft-deleted product is not removed from the database. Instead, it is hidden from normal users by adding a condition such as:

```sql
deletedAt IS NULL
```

## My hypothesis

The Christmas special offer of 2014 is probably an old product that still exists in the database, but is hidden from the frontend because its `deletedAt` value is not `null`.

If I can bypass the condition that hides deleted products, I may be able to make the deleted Christmas product visible again.

## Important endpoint

```http
GET /rest/products/search?q=
```

This endpoint was interesting because:

* It accepts user-controlled input through the `q` parameter.
* It returns product data from the database.
* Previous SQL injection testing showed that this endpoint can be manipulated.
* The response contains product fields such as `id`, `name`, `description`, `price`, and `deletedAt`.

## Backend logic inferred

The backend query likely behaves like this:

```sql
SELECT * FROM Products
WHERE (
  (name LIKE '%USER_INPUT%' OR description LIKE '%USER_INPUT%')
  AND deletedAt IS NULL
)
ORDER BY name
```

The important part is:

```sql
AND deletedAt IS NULL
```

This condition hides deleted or unavailable products from customers.

## Testing steps

### 1. Inspect normal product search

I first sent a normal product search request to Burp Repeater:

```http
GET /rest/products/search?q=
```

The response returned products where:

```json
"deletedAt": null
```

This confirmed that normal product search only shows active products.

### 2. Focus on the hidden product logic

The challenge hints mentioned that the product is unavailable or deleted. Based on the `deletedAt` field, I assumed the application was hiding old products using a SQL condition.

The goal became:

```text
Make deleted products visible again.
```

### 3. Craft a SQL injection payload

Because the input is inserted inside a `LIKE '%...%'` expression, I needed to break out of the string and comment out the remaining SQL condition.

A payload shape for this type of query is:

```text
'))--
```

Example request:

```http
GET /rest/products/search?q='))--
```

The purpose of the payload is to:

1. Close the string.
2. Close the parentheses.
3. Comment out the rest of the query, including `AND deletedAt IS NULL`.

### 4. Identify the deleted Christmas product

After making deleted products visible, I searched the response for keywords such as:

```text
Christmas
2014
```

The target product was identified from the JSON response.

Important fields to collect:

```text
id
name
description
price
deletedAt
```

### 5. Add the hidden product to the basket

The product could not be ordered through the normal frontend because it was hidden or unavailable.

Instead, the product had to be added to the basket through the backend API.

The important idea was:

```text
Use the product ID from the hidden product and submit it directly to the basket API.
```

A typical basket item request has this kind of structure:

```http
POST /api/BasketItems/
Content-Type: application/json
```

```json
{
  "ProductId": "<hidden_product_id>",
  "BasketId": "<current_basket_id>",
  "quantity": 1
}
```

After adding the product to the basket, I triggered checkout to complete the challenge.

## Result

The challenge was solved after:

1. Making deleted products visible through SQL injection.
2. Identifying the Christmas special product from 2014.
3. Adding the hidden product directly to the basket through the API.
4. Completing checkout.

## Why it worked

The application relied on a SQL condition to hide deleted products:

```sql
deletedAt IS NULL
```

Because the product search endpoint was vulnerable to SQL injection, it was possible to alter the query logic and bypass this condition.

The deleted product still existed in the database, so once the filter was bypassed, its product ID could be recovered and used in an API request.

## Root cause

The root cause is unsafe SQL query construction in the product search endpoint.

Additional design issues:

* Deleted products were only hidden through query filtering.
* The frontend did not show deleted products, but the backend still allowed them to be referenced.
* The basket API accepted a product ID directly.
* Server-side business rules did not properly prevent ordering unavailable products.

## Impact in a real application

In a real application, this type of vulnerability could allow an attacker to:

* Access hidden or deleted products.
* Order products that should no longer be available.
* Bypass business rules enforced only by the frontend.
* Retrieve internal product data.
* Abuse discontinued discounts or special offers.
* Manipulate inventory or pricing logic.

## Remediation

Recommended fixes:

* Use parameterized queries or prepared statements.
* Do not concatenate user input into SQL queries.
* Enforce product availability checks server-side before adding items to the basket.
* Validate that a product is active before checkout.
* Do not rely only on frontend visibility to protect business logic.
* Apply proper authorization and business rule validation on all API endpoints.
* Return generic errors instead of exposing SQL behavior.
* Add automated tests for SQL injection and unavailable product ordering.

## What I learned

This challenge taught me that hidden data is not necessarily protected.

If an application uses soft deletion, deleted records may still exist in the database and can sometimes be accessed if query logic is bypassed.

The important lesson is that security must be enforced server-side, not only through frontend visibility.

## Mistake or learning point

At first, I focused on viewing normal products. The key observation was noticing the `deletedAt` field.

Once I understood that `deletedAt = null` means visible product, I realized that `deletedAt` with a date probably means hidden or deleted product.

## Memory rule

```text
deletedAt = null  → visible product
deletedAt = date  → deleted or hidden product
```

For this challenge:

```text
Bypass deletedAt IS NULL → reveal hidden product → add it through the API → checkout
```

## Related patterns

* SQL injection in search functionality
* Soft-delete bypass
* API business logic bypass
* Hidden product enumeration
* Frontend restriction bypass
