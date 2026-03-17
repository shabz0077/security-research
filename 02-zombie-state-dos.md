# Finding 02: Zombie State DoS — Permanent Resource Lock via State Machine Flaw

**Severity:** CRITICAL  
**CVSS Score:** 8.7  
**CVSS Vector:** AV:N/AC:L/PR:L/UI:N/S:C/C:N/I:N/A:H  
**Category:** Business Logic / State Management  
**OWASP API:** API6:2023 — Unrestricted Access to Sensitive Business Flows  
**Status:** Reported and patched  
**Phase:** Phase 1

---

## Summary

After successfully reserving a locker without payment (Finding 01), I discovered that the
reservation state machine had a fatal paradox. A locker reserved without payment would
enter a "busy" state in the database, but the unreserve endpoint would reject release
attempts with "Locker not in use." The locker was permanently bricked with no admin
recovery path — creating a Denial of Service condition I called the Zombie State.

---

## How I Found It

After the free reservation from Finding 01 went through, I wanted to see if I could release
the locker. I sent a request to the unreserve endpoint:

```http
POST /api/v1/[CONTROLLER-ID]/locker/[LOCKER-UUID]/unreserve_locker/ HTTP/1.1
Host: [REDACTED]
Authorization: Bearer [USER-TOKEN]
Content-Type: application/json
```

I expected either a success response or a "you don't own this locker" error. Instead I got:

```json
HTTP/1.1 400 Bad Request

{"message": "Locker not in use"}
```

But the locker was definitely in use — it was showing as "busy" in the locker list endpoint.
So the database said busy, but the unreserve logic said not in use. That's a state machine
contradiction — the locker was stuck in a state that the application couldn't resolve.

I checked the locker status again after waiting — still busy. I tried the unreserve request
again — still rejected. I waited longer. Same result. The locker was permanently stuck with
no way to release it through the normal application flow.

This meant one reservation request could permanently destroy one unit of inventory.

---

## Root Cause

The reservation flow had two separate state checks that were out of sync:

The database marked the locker as "in use" immediately when the reservation POST succeeded.
The unreserve endpoint checked a different condition — likely whether a payment-confirmed
reservation existed — before allowing release. Since no payment was made, the unreserve
logic found no valid reservation to cancel, even though the database showed the locker
as occupied.

The two states could never reconcile. No admin interface existed to manually reset
individual locker states.

---

## Impact

- Each free reservation permanently removes one locker from service
- No recovery without direct database intervention
- Chained with automation (see Finding 08), this could take all inventory offline
- Business loses revenue from permanently unavailable units

---

## CVSS Breakdown

| Metric | Value | Reason |
|--------|-------|--------|
| Attack Vector | Network | Remote API call |
| Attack Complexity | Low | Single POST request |
| Privileges Required | Low | Valid user account |
| User Interaction | None | No victim interaction |
| Scope | Changed | Affects all users of the service, not just the attacker |
| Confidentiality | None | No data exposed |
| Integrity | None | No data modified |
| Availability | High | Permanent removal of inventory from service |

---

## Remediation

- Implement a unified reservation state — a single source of truth that both reserve and unreserve endpoints reference
- Payment validation should be a prerequisite before any locker state change is committed to the database
- Add an admin endpoint or scheduled cleanup job that can force-reset locker states
- If payment is not confirmed within the hold window, auto-release must trigger regardless of the reservation path used
