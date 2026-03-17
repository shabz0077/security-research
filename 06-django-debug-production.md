# Finding 06: Django Debug Mode Enabled in Production Environment

**Severity:** HIGH  
**CVSS Score:** 7.2  
**CVSS Vector:** AV:N/AC:L/PR:N/UI:N/S:U/C:H/I:N/A:N  
**Category:** Security Misconfiguration / Information Disclosure  
**OWASP API:** API8:2023 — Security Misconfiguration  
**Status:** Reported  
**Phase:** Phase 1

---

## Summary

The production Django application was running with DEBUG=True. Navigating to any invalid
or non-existent URL returned a full Django debug page showing the complete URL routing
table, internal module paths, stack traces, and environment details. This gave me a
readable map of the entire backend architecture.

---

## How I Found It

While exploring the application I accidentally navigated to a URL path that didn't exist.
Instead of a standard 404, I got the yellow Django debug page — immediately recognisable
to anyone who has worked with Django development environments.

The debug page showed:

- Complete list of all URL patterns in the application
- Internal Python module paths revealing the project structure
- Stack traces with file paths showing where code lived on the server
- Environment configuration details

This is a well-known Django misconfiguration — `DEBUG = True` is the default development
setting and should always be set to `False` before any production deployment. It is one
of the first things the Django documentation warns about.

---

## How It Contributed to the Attack Chain

On its own this is an information disclosure finding. In combination with the exposed
Swagger UI (Finding 05) it gave a complete picture of the application:

- Swagger UI showed what endpoints existed and what parameters they accepted
- Django debug pages showed how the backend code was structured internally
- Together they answered both "what can I call" and "how does it work"

When I was fuzzing the `code` parameter in Phase 2 (Finding 08), having the internal
URL structure from the debug pages helped me understand what backend logic might be
processing that parameter.

---

## CVSS Breakdown

| Metric | Value | Reason |
|--------|-------|--------|
| Attack Vector | Network | Any invalid URL triggers debug page |
| Attack Complexity | Low | No special knowledge required |
| Privileges Required | None | Unauthenticated |
| User Interaction | None | No victim required |
| Scope | Unchanged | Information disclosure within application |
| Confidentiality | High | Internal architecture fully exposed |
| Integrity | None | Read-only |
| Availability | None | No service impact |

---

## Remediation

- Set `DEBUG = False` in all production Django settings
- Use separate settings files for development and production — never share the same config
- Configure a custom 404 and 500 error page that returns no internal information
- Review all environment variables before deployment using a pre-deployment checklist
