# Finding 08: Application-Level DoS via Temporary Hold Feature Abuse

**Severity:** CRITICAL  
**CVSS Score:** 9.0  
**CVSS Vector:** AV:N/AC:L/PR:L/UI:N/S:C/C:N/I:N/A:H  
**Category:** Business Logic / Unrestricted Resource Consumption  
**OWASP API:** API6:2023 — Unrestricted Access to Sensitive Business Flows  
**Status:** Reported  
**Phase:** Phase 2 — Post-patch independent discovery

---

## Summary

After Phase 1 findings were patched, I continued manual testing on the endpoints. Through
parameter fuzzing I discovered a hidden backend parameter that could trigger the
application's legitimate 5-minute temporary hold mechanism. While the mechanism itself
was correctly designed — lockers auto-release after 5 minutes if payment is not completed
— there was no rate limiting, no per-user concurrent hold cap, and no detection for
automated abuse. By scripting requests to continuously re-trigger holds before the
auto-release timer fired, I kept 100% of inventory in an assigned state permanently.

---

## How I Found It — The Full Story

Phase 1 had been patched. I was doing post-patch manual testing on the reservation
endpoint, specifically testing whether the discount and promo code features had any
interesting behaviour.

I started by injecting a discount parameter I had noticed the UI was sending in certain
flows. The server responded asking for a `promo_code` field. I started fuzzing that
field — valid promo codes, invalid strings, SQLi payloads, random values.

After several attempts the server threw a different error. Instead of the usual
`promo_code error` it returned something I hadn't seen before: `code: error`.

That was interesting. The error message was referencing a field called `code` — not
`promo_code`. The server was telling me that a completely separate parameter called
`code` existed in the backend logic and I had somehow triggered a validation error for it.

I started injecting the `code` parameter directly with various values. When I sent
`code: FREE` the server returned:

```json
HTTP/1.1 200 OK

{"message": "Locker assigned successfully"}
```

My first thought was payment bypass — same as Finding 01. I reported it to the team.

The team came back and told me it was not a payment bypass. They explained the application
used a BookMyShow-style temporary hold mechanism — when a user starts the booking flow,
the locker enters an "assigned" state for 5 minutes while payment is being processed. If
payment is not completed within 5 minutes, a cleanup job runs and releases the locker
automatically. The `code: FREE` injection was triggering this hold mechanism, not bypassing
payment. After 5 minutes the locker would be released.

They were right. I re-tested and confirmed the locker released after 5 minutes.

I did not stop there.

---

## The Actual Vulnerability

The hold mechanism itself was correctly designed. The problem was what surrounded it:

- **No rate limiting** on the reservation endpoint
- **No per-user concurrent hold limit** — one user could hold every locker simultaneously
- **No detection** for automated abuse of the hold feature
- **Predictable cleanup timing** — the 5-minute release window was consistent and known

If I sent one request every 4 minutes and 50 seconds — just before the cleanup job ran —
the locker would re-enter the assigned state before it had a chance to release. Repeat
this for all 8 lockers simultaneously and the entire inventory stays permanently assigned
with no legitimate customer ever able to book.

The feature was a door. The missing controls were the lock on that door.

---

## Proof of Concept

I wrote a Python script to demonstrate this. The script:

1. Iterates through all locker UUIDs
2. Sends a hold request to each one
3. Waits 290 seconds (4 minutes 50 seconds — just under the 5-minute cleanup window)
4. Repeats indefinitely
5. Automatically refreshes authentication tokens to maintain valid sessions

The script ran continuously keeping all 8 lockers in assigned state. No legitimate user
could complete a booking during the script's execution. The service was effectively
offline for customers while the script ran.

```python
# Sanitized PoC — demonstrates the attack logic
# All credentials, endpoints, and UUIDs have been removed

import requests
import time

# Configuration (sanitized)
BASE_URL = "[REDACTED]"
CONTROLLER_ID = "[REDACTED]"
TARGET_LOCKERS = {
    "[LOCKER-UUID-1]": "S1",
    "[LOCKER-UUID-2]": "S2",
    # ... all lockers
}

def reserve_locker(uuid, name, token):
    url = f"{BASE_URL}/api/v1/{CONTROLLER_ID}/locker/{uuid}/reserve_locker/"
    headers = {"Authorization": f"Bearer {token}", "Content-Type": "application/json"}
    payload = {"hours": 10, "code": "FREE"}  # Triggers 5-min hold

    response = requests.post(url, json=payload, headers=headers, timeout=5)
    if response.status_code == 200:
        print(f"[SUCCESS] {name} -> Assigned state triggered")
        return True
    return False

def dos_loop(token):
    cycle = 1
    while True:
        print(f"\n--- Cycle {cycle} ---")
        for uuid, name in TARGET_LOCKERS.items():
            reserve_locker(uuid, name, token)
            time.sleep(0.5)  # Small delay between requests

        # Wait 290 seconds — just under the 5-minute auto-release window
        # This re-triggers holds before cleanup job fires
        print("Waiting 290s before next cycle...")
        time.sleep(290)
        cycle += 1
```

---

## Why I Initially Misread This

This is worth documenting honestly because it shows how finding re-evaluation works
in practice.

When I first saw the 200 OK response after injecting `code: FREE` I assumed payment
bypass — same pattern as Finding 01. I reported it as such. The vendor correctly pushed
back and explained the temporary hold mechanism.

A less persistent tester might have accepted that correction and moved on. The key step
was asking the follow-up question: "OK, it's not a payment bypass — but what else can I
do with a mechanism that holds a resource for 5 minutes with no controls around it?"

That question led to the actual finding. The initial misread was not wasted — it was
the path to the real vulnerability.

---

## Impact

- 100% of bookable inventory can be kept in assigned state permanently by a single user
- No legitimate customer can complete a booking while the attack is running
- The attack requires only a valid user account and an internet connection
- Revenue loss is total for the duration of the attack — zero bookings possible

---

## CVSS Breakdown

| Metric | Value | Reason |
|--------|-------|--------|
| Attack Vector | Network | Remote API, no physical access needed |
| Attack Complexity | Low | Simple repeated POST requests |
| Privileges Required | Low | Valid user account required |
| User Interaction | None | Fully automated |
| Scope | Changed | All users of the service affected |
| Confidentiality | None | No data exposed |
| Integrity | None | No data modified |
| Availability | High | 100% service unavailability |

---

## Remediation

- Implement per-user concurrent hold limit — one active hold per user at any time
- Add rate limiting on the reservation endpoint — maximum N requests per minute per account
- Monitor for users holding multiple lockers simultaneously and flag for review
- Implement detection for automated hold cycling patterns
- Consider requiring CAPTCHA or additional verification for bulk hold attempts
- The temporary hold feature itself does not need to change — only the missing controls around it
