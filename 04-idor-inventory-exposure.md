# Finding 04: IDOR — Full Inventory Data Exposure Without Authentication

**Severity:** HIGH  
**CVSS Score:** 7.5  
**CVSS Vector:** AV:N/AC:L/PR:N/UI:N/S:U/C:H/I:N/A:N  
**Category:** Insecure Direct Object Reference (IDOR)  
**OWASP API:** API1:2023 — Broken Object Level Authorization  
**Status:** Reported  
**Phase:** Phase 1

---

## Summary

The locker listing endpoint returned full JSON data for all lockers in the facility —
including internal UUIDs, controller IDs, and real-time availability status — without
requiring any authentication. This was the reconnaissance foundation that made all
targeted attacks in this engagement possible.

---

## How I Found It

After accessing the Swagger UI with my Bearer token, I could see a locker list endpoint.
I tested it without the Authorization header first — just to see what happened.

```http
GET /api/v1/[CONTROLLER-ID]/locker/ HTTP/1.1
Host: [REDACTED]
```

Response returned full inventory data — no authentication required:

```json
[
  {
    "id": "[LOCKER-UUID]",
    "name": "S1",
    "status": true,
    "controller": "[CONTROLLER-UUID]",
    "locker_type": "small"
  },
  ...
]
```

This gave me every locker UUID, real-time availability (true = available), and the
controller ID needed to construct valid API requests. Without this, targeted attacks
against specific lockers would have required guessing UUIDs.

---

## Why This Finding Matters Beyond Its Own Severity

In isolation this is a High — information disclosure, no direct exploitation. But in the
context of this engagement it was the enabler for everything else. The CVSS score rates
the finding standalone. The real-world impact is multiplied because the exposed UUIDs
were used directly in Findings 01, 02, 03, and 08.

A finding that scores 7.5 in isolation can score 9.3 in a chain. That gap is why manual
testing finds more than automated scanners — scanners score findings individually,
attackers chain them.

---

## CVSS Breakdown

| Metric | Value | Reason |
|--------|-------|--------|
| Attack Vector | Network | Publicly accessible endpoint |
| Attack Complexity | Low | Simple GET request |
| Privileges Required | None | No authentication required |
| User Interaction | None | No victim interaction |
| Scope | Unchanged | Stays within application boundary |
| Confidentiality | High | All internal resource identifiers exposed |
| Integrity | None | Read-only |
| Availability | None | No service impact |

---

## Remediation

- Require authentication on all inventory endpoints
- Return only lockers relevant to the authenticated user's session
- Mask internal UUIDs from API responses — use session-scoped tokens instead of permanent database IDs
- Apply rate limiting to list endpoints to prevent automated enumeration
