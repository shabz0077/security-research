# Finding 07: Weak OTP Authentication — No Rate Limiting or Device Binding

**Severity:** MEDIUM  
**CVSS Score:** 6.5  
**CVSS Vector:** AV:N/AC:L/PR:N/UI:N/S:U/C:L/I:L/A:N  
**Category:** Broken Authentication  
**OWASP API:** API2:2023 — Broken Authentication  
**Status:** Reported  
**Phase:** Phase 1

---

## Summary

The OTP-based authentication endpoint had no rate limiting, no attempt throttling, and no
device binding. Bearer tokens were issued after simple OTP verification with no additional
checks. I demonstrated automated token generation using a Python script — showing that
user impersonation at scale was possible without any brute force protection.

---

## How I Found It

During reconnaissance I was looking at the authentication flow through Burp Suite. The
login process was:

1. User enters phone number
2. Server sends OTP
3. User submits OTP
4. Server returns Bearer token

I noticed there was no rate limiting on the OTP submission endpoint — I could submit
incorrect OTPs repeatedly with no lockout or delay. There was also no device fingerprinting
or binding — the same OTP could be used from any device or IP address.

I wrote a simple Python script to automate the token generation process and confirmed
that valid tokens could be obtained programmatically with no friction. The script could
request tokens for any phone number that received an OTP.

---

## Why This Is Medium and Not Higher

The impact per successful exploit is limited to one user account. You need the OTP to
complete the flow — you can't brute force the 6-digit OTP fast enough before it expires
in a normal scenario. The real risk is at scale — if an attacker had access to a list of
phone numbers, or could intercept SMS, the lack of device binding means tokens could
be generated from anywhere.

The finding is Medium in isolation but gains significance in the context of this engagement
because valid Bearer tokens were the prerequisite for all other findings.

---

## CVSS Breakdown

| Metric | Value | Reason |
|--------|-------|--------|
| Attack Vector | Network | Remote API |
| Attack Complexity | Low | Simple automated requests |
| Privileges Required | None | No prior authentication |
| User Interaction | None | No victim interaction required |
| Scope | Unchanged | Single account scope |
| Confidentiality | Low | Access to one user account's data |
| Integrity | Low | Can perform actions as the user |
| Availability | None | No service disruption |

---

## Remediation

- Implement rate limiting on OTP submission — maximum 3-5 attempts before temporary lockout
- Add exponential backoff after failed attempts
- Implement device binding — tie tokens to device fingerprint or IP
- Set short OTP expiry windows (2-3 minutes maximum)
- Alert on unusual OTP request patterns from single IPs or phone numbers
