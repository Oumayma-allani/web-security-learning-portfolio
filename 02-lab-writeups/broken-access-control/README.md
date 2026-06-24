# Broken Access Control Writeups

This folder contains my writeups for legal Broken Access Control labs completed on OWASP Juice Shop.

The goal is not only to document final requests, but to explain:

* what I observed
* how I identified access control weaknesses
* what fields or IDs were controlled by the client
* what mistakes I made
* why the vulnerability worked
* what the real-world impact would be
* how to remediate it

## Covered patterns

* IDOR and basket manipulation
* HTTP Parameter Pollution
* Client-controlled `UserId`
* Client-controlled `author` field
* Missing server-side ownership checks
* API authorization bypass
