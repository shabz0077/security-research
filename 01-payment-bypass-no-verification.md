# Finding 01: Payment Bypass — Reservation Without Payment Verification

**Severity:** CRITICAL  
**CVSS Score:** 9.1  
**CVSS Vector:** AV:N/AC:L/PR:L/UI:N/S:C/C:N/I:H/A:H  
**Category:** Business Logic / Broken Access Control  
**OWASP API:** API3:2023 — Broken Object Property Level Authorization  
**Status:** Reported and patched  
**Phase:** Phase 1

---

## Summary

The locker reservation endpoint accepted booking requests from authenticated users without
verifying whether the user had completed payment. Any user with a valid Bearer token could
reserve a locker for free by sending a direct API request, completely bypassing the
third-party payment gateway.

---

## How I Found It

My starting point was the exposed Swagger UI (see Finding 05). Once I had access to the
full API documentation using a regular user Bearer token, I could see the complete list of
endpoints including `/reserve_locker/`.

I noticed that the Swagger UI showed the endpoint only required `hours` as a parameter.
There was no `payment_id` or `transaction_id` field in the request schema. That immediately
looked wrong to me — how does the server know if you paid?

I intercepted a normal reservation flow using Burp Suite and watched what happened when a
real user booked a locker through the app. The app first called the payment gateway, got a
payment confirmation, then called `/reserve_locker/`. But the reservation endpoint itself
never passed the payment confirmation ID to the backend — it just sent `hours` and the
locker ID.

So I skipped the payment step entirely and sent the reservation request directly:

```http
POST /api/v1/[CONTROLLER-ID]/locker/[LOCKER-UUID]/reserve_locker/ HTTP/1.1
Host: [REDACTED]
Authorization: Bearer [USER-TOKEN]
Content-Type: application/json

{
    "hours": 10
}
```

The server responded:

```json
HTTP/1.1 200 OK

{"message": "Locker assigned successfully"}
```

No payment. No verification. The locker was reserved.

---

## Root Cause

The backend treated authentication (valid Bearer token) as authorization to reserve.
It never checked whether a payment had been initiated or completed for that reservation.
The payment gateway was only integrated on the frontend — the backend had no server-side
payment state validation before confirming a booking.

---

## Impact

- Any registered user could reserve lockers at zero cost
- Financial loss for the platform on every exploited reservation
- Inventory locked by users who never paid, reducing availability for legitimate customers

---

## CVSS Breakdown

| Metric | Value | Reason |
|--------|-------|--------|
| Attack Vector | Network | Exploitable via internet from any location |
| Attack Complexity | Low | Straightforward POST request, no special conditions |
| Privileges Required | Low | Valid user account required |
| User Interaction | None | No victim interaction needed |
| Scope | Changed | Impacts financial systems outside the application boundary |
| Confidentiality | None | No data exposed |
| Integrity | High | Fraudulent reservation record written to database |
| Availability | High | Inventory reduced for legitimate users |

---

## Remediation

- Implement server-side payment verification before confirming any reservation
- Generate a server-side payment session token that must be passed and validated in the reservation request
- Never rely solely on frontend payment flow — backend must independently confirm payment status with the payment gateway before writing reservation records
