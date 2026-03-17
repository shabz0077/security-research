# Finding 05: Exposed Swagger UI — Unauthenticated Production API Documentation

**Severity:** HIGH  
**CVSS Score:** 7.5  
**CVSS Vector:** AV:N/AC:L/PR:N/UI:N/S:U/C:H/I:N/A:N  
**Category:** Security Misconfiguration / Information Disclosure  
**OWASP API:** API9:2023 — Improper Inventory Management  
**Status:** Reported  
**Phase:** Phase 1

---

## Summary

The production application had its Swagger UI publicly accessible without authentication.
This gave anyone the complete interactive API documentation — every endpoint, every
parameter, every authentication mechanism. It was the starting point of this entire
engagement and the reason the attack chain was possible.

---

## How I Found It

I was manually exploring the application and tried navigating to common developer paths.
The Swagger UI was sitting at a predictable URL with no login required.

When I opened it I found complete interactive documentation for the entire API — locker
operations, controller management, payment flows, authentication endpoints. Everything
was documented with parameters, request examples, and response schemas.

The critical part was the authorization section. The Swagger UI had a standard "Authorize"
button that accepted Bearer tokens. I pasted my normal user Bearer token — the same one
the mobile app uses — and the entire API documentation became interactive. I could send
real requests directly from the documentation interface.

This is where I found the `/reserve_locker/` endpoint schema that showed no payment
parameter was required (Finding 01), and where I discovered the `/set_password/` endpoint
that later led to Finding 03.

---

## What Made This More Than Just Information Disclosure

Most Swagger UI exposures are treated as medium severity — you see the docs but you still
need credentials to do anything. This one was different because:

1. A normal user token was accepted on dev-level endpoints
2. The Django debug mode (Finding 06) was also active — so error responses from the
   Swagger UI were leaking internal code structure alongside the documentation
3. The combination gave a complete attack surface map with internal implementation details

The Swagger UI alone I would rate as High. Combined with Django debug mode it was
effectively a full system blueprint handed to any user who navigated to the right URL.

---

## CVSS Breakdown

| Metric | Value | Reason |
|--------|-------|--------|
| Attack Vector | Network | Publicly accessible URL |
| Attack Complexity | Low | Just navigate to the path |
| Privileges Required | None | No authentication to view docs |
| User Interaction | None | No victim required |
| Scope | Unchanged | Information disclosure only |
| Confidentiality | High | Complete API structure exposed |
| Integrity | None | Read-only |
| Availability | None | No service impact |

---

## Remediation

- Disable Swagger UI entirely in production — it is a development tool only
- If API documentation must be accessible, require authentication and restrict to internal networks or specific IP ranges
- Never deploy debug-enabled documentation interfaces to production environments
