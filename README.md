# Security Research — Sabarinath R

Vulnerability findings from authorized penetration testing engagements.
Company names, domains, endpoints, and credentials sanitized with vendor permission.

---

## Findings Index

| # | Title | Severity | CVSS | Phase |
|---|-------|----------|------|-------|
| 1 | [Payment Bypass — Reservation Without Payment Verification](./findings/01-payment-bypass-no-verification.md) | CRITICAL | 9.1 | Phase 1 |
| 2 | [Zombie State DoS — Permanent Resource Lock](./findings/02-zombie-state-dos.md) | CRITICAL | 8.7 | Phase 1 |
| 3 | [Unauthorized Physical Access via Password Reset](./findings/03-physical-access-password-reset.md) | CRITICAL | 9.3 | Phase 1 |
| 4 | [IDOR — Full Inventory Exposure Without Authentication](./findings/04-idor-inventory-exposure.md) | HIGH | 7.5 | Phase 1 |
| 5 | [Exposed Swagger UI — Unauthenticated Production Access](./findings/05-exposed-swagger-ui.md) | HIGH | 7.5 | Phase 1 |
| 6 | [Django Debug Mode Enabled in Production](./findings/06-django-debug-production.md) | HIGH | 7.2 | Phase 1 |
| 7 | [Weak OTP Auth — No Rate Limiting](./findings/07-weak-otp-auth.md) | MEDIUM | 6.5 | Phase 1 |
| 8 | [Application-Level DoS via Temporary Hold Feature Abuse](./findings/08-appdos-temporary-hold-abuse.md) | CRITICAL | 9.0 | Phase 2 |

---

## Target Context

**Type:** IoT-integrated locker management platform (mobile app + REST API)  
**Stack:** Django/Python backend, REST API, third-party payment gateway  
**Assessment:** Black-box web application and API penetration test  
**Authorization:** Verbal authorization granted by vendor. Findings disclosed prior to publication.  
**Phases:** Two separate testing phases — initial engagement and post-patch follow-up testing

---

## Phase 1 Attack Chain

Discovered through manual exploration starting from an exposed Swagger UI:

```
Exposed Swagger UI (no authentication required)
            ↓
Normal user Bearer token accepted on developer-level API docs
            ↓
Django debug mode leaking internal code structure and URL routing
            ↓
Reservation endpoint accepts booking request without payment verification
            ↓
Zombie State created — locker permanently busy, unreserve endpoint rejects it
            ↓
Password reset works on zombie locker → Physical access without completing payment
```

## Phase 2 Attack Chain

Discovered independently during post-patch manual testing:

```
Manual endpoint fuzzing after Phase 1 patch applied
            ↓
Discount parameter injection → server requests promo_code field
            ↓
Continued fuzzing with promo codes, SQLi payloads, random strings
            ↓
Server throws "code: error" → hidden backend parameter confirmed
            ↓
Injected code:FREE → 200 OK → locker enters 5-minute assigned/hold state
            ↓
Initially misread as payment bypass → reported → vendor correctly clarified
it is a legitimate BookMyShow-style temporary hold, not a payment bypass
            ↓
Key insight: temp hold + no rate limit + no per-user cap = permanent DoS
            ↓
Python script cycling all lockers every 290 seconds beats the 5-min cleanup job
→ 100% of inventory permanently in assigned state → service fully unavailable
```

---

## Tools Used

- Burp Suite (HTTP interception, parameter analysis, manual testing)
- Python (PoC automation, token refresh logic, DoS loop scripting)
- Nmap (network reconnaissance)
- Wireshark (traffic analysis)

---

## Contact

**LinkedIn:** [linkedin.com/in/sabarinath-r](https://linkedin.com/in/sabarinath-r)  
**GitHub:** [github.com/shabz0077](https://github.com/shabz0077)
