# Finding 03: Unauthorized Physical Access via Password Reset on Zombie State Locker

**Severity:** CRITICAL  
**CVSS Score:** 9.3  
**CVSS Vector:** AV:N/AC:L/PR:L/UI:N/S:C/C:H/I:H/A:H  
**Category:** Broken Access Control  
**OWASP API:** API1:2023 — Broken Object Level Authorization  
**Status:** Reported and patched  
**Phase:** Phase 1

---

## Summary

After creating a Zombie State locker (Finding 02), I tested whether other endpoints still
worked on the locked resource. The password reset endpoint accepted requests on zombie
state lockers and allowed me to set a custom PIN — granting physical access to the locker
without ever completing payment. This is the highest severity finding in the engagement
because it crosses the digital-to-physical boundary.

---

## How I Found It

At this point I had a locker stuck in zombie state. I started testing what else I could do
with it. The Swagger UI (Finding 05) showed a password reset endpoint. My thinking was
simple: if the locker is in a broken state, maybe other state checks are also broken.

I sent a password reset request to the zombie locker:

```http
POST /api/v1/[CONTROLLER-ID]/locker/[LOCKER-UUID]/set_password/ HTTP/1.1
Host: [REDACTED]
Authorization: Bearer [USER-TOKEN]
Content-Type: application/json

{
    "password": "[REDACTED]",
    "pin": "[REDACTED]"
}
```

Response:

```json
HTTP/1.1 200 OK

{"message": "Locker pass set successfully"}
```

The locker accepted the new PIN. I now had a locker that was physically accessible using
the PIN I just set — without completing any payment. The locker was a physical unit. Anyone
with the PIN could walk up and open it.

---

## Why This Is The Most Severe Finding

Every other finding in this engagement was digital — financial loss, service disruption,
data exposure. This one has physical world consequences. A digital API exploit translates
directly into unauthorized access to a physical storage unit. That's why all three CIA
components are High:

- **Confidentiality:** Attacker can access whatever is stored inside the locker
- **Integrity:** Attacker has overwritten the legitimate owner's PIN
- **Availability:** Legitimate owner is now locked out of their own reservation

---

## Root Cause

The password reset endpoint checked authentication (valid Bearer token) but did not check
the reservation state or payment status of the locker before allowing PIN modification.
Each endpoint was validating independently without checking the overall session state.

---

## CVSS Breakdown

| Metric | Value | Reason |
|--------|-------|--------|
| Attack Vector | Network | API call from anywhere |
| Attack Complexity | Low | Straightforward request |
| Privileges Required | Low | Valid user account |
| User Interaction | None | No victim interaction |
| Scope | Changed | Digital exploit with physical world impact |
| Confidentiality | High | Physical contents of locker accessible |
| Integrity | High | PIN overwritten, access control record modified |
| Availability | High | Legitimate owner locked out |

---

## Remediation

- Password reset endpoint must verify that the requesting user has an active, payment-confirmed reservation on the target locker
- Implement server-side reservation ownership checks on all locker operation endpoints — not just authentication
- Physical access operations (PIN set, PIN change, unlock) should require the highest level of authorization validation
