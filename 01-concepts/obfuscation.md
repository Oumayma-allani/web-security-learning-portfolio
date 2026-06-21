# Obfuscation

## Simple definition

Obfuscation means making data harder to read or detect while keeping its meaning after decoding or interpretation.

In web security, attackers may use obfuscation to hide suspicious input from weak filters, WAF rules, or client-side checks.

## Why it matters

Some applications block obvious payloads but fail when the same payload is encoded or represented differently.

The important idea is:

```text
Filter sees encoded input
Backend/browser decodes it
Dangerous meaning appears later
```

## Common forms

### URL encoding

URL encoding changes special characters into `%XX` format.

Example idea:

```text
space  -> %20
'      -> %27
<      -> %3C
>      -> %3E
```

Normal use:

- Avoid URL ambiguity.
- Safely transmit special characters in URLs.

Security risk:

- Attackers may encode characters to bypass weak filters.
- The server often decodes the input before processing it.

### HTML encoding

HTML encoding represents characters using HTML entities.

Example idea:

```text
<  -> &lt;
>  -> &gt;
"  -> &quot;
'  -> &#x27;
```

Normal use:

- Safely display special characters in HTML.

Security risk:

- Encoded characters may be decoded by the browser when parsing HTML.
- Weak filters may fail if they only search for raw dangerous characters.

### XML encoding

XML encoding uses XML entities to represent characters inside XML documents.

This matters when the application receives XML input and decodes it before passing values to backend logic.



## Defensive lessons

- Do not rely only on blacklists or WAF filters.
- Decode and normalize input before validation.
- Use safe APIs such as parameterized queries.
- Apply output encoding based on context.
- Validate input server-side.
- Treat encoded input as potentially dangerous after decoding.

## Beginner mistakes to avoid

- Thinking that encoding makes a payload safe.
- Testing only raw input and ignoring encoded versions.
- Forgetting that browsers and servers may decode input automatically.
- Relying on WAF behavior instead of understanding the root cause.

## Related Topics

- Context-specific encoding.
- Canonicalization and input normalization.
- Difference between encoding, encryption, and hashing.
- Why weak filters can miss obfuscated input.
