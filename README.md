# Web Application Attack & Defense — DVWA + ModSecurity (OWASP CRS)

## Overview
Exploited SQL Injection, Cross-Site Scripting and File Upload vulnerabilities in DVWA, then deployed and tuned a ModSecurity web application firewall running the OWASP Core Rule Set to block the same attacks. Documented the measurable difference between detection-only and active blocking.

**Environment:** Kali Linux, DVWA (native install), Apache, ModSecurity 2 with OWASP CRS, Metasploit (msfvenom), ELK Stack for log correlation.

> DVWA is intentionally vulnerable software built for this exact purpose.
> No production or third-party systems were targeted.

## What I Did

### Attack Phase
1. Set DVWA's security level to Low to reliably trigger its vulnerabilities.
   <img width="623" height="720" alt="IMG_8477" src="https://github.com/user-attachments/assets/34ea4053-6326-4f1b-a706-857eefa477a1" />
2. Executed a classic SQL Injection payload (`1' OR '1'='1`) against the
SQLi module, successfully returning every user record instead of one.
3. Executed a reflected XSS payload against the XSS module, triggering script execution in the browser.
4. Generated a PHP payload with `msfvenom`, uploaded it through DVWA's File Upload page, and caught a reverse shell with Metasploit's `multi/handler`, demonstrating remote code execution via an unrestricted file upload.

### Defense Phase
5. Installed ModSecurity with the OWASP Core Rule Set, initially running in `DetectionOnly` mode (logs, doesn't block). This is why the attacks above succeeded, nothing was actively enforced yet.
6. Switched `SecRuleEngine` to `On` and reran every payload.
7. Confirmed each attack now returned HTTP 403 Forbidden before reaching DVWA's application code.

## Before vs After

| Attack | DetectionOnly | SecRuleEngine On |
|---|---|---|
| SQL Injection | Succeeded, logged by CRS rule 942100 | Blocked (403) |
| Reflected XSS | Succeeded, logged by CRS rule 941110 | Blocked (403) |
| Malicious file upload + shell | Succeeded, SYSTEM-level shell obtained | Blocked (403) |

## Key Finding
A WAF in detection mode provides visibility, not protection. Every attack
that succeeded in this lab did so specifically because blocking hadn't been enabled yet, the same payloads failed once `SecRuleEngine` was switched to `On`. This is a real-world configuration mistake: teams often leave a WAF in DetectionOnly "for tuning" far longer than intended.

## Evidence
- ModSecurity config showing both modes (`DetectionOnly` → `On`)
  <img width="858" height="720" alt="Evidence 1" src="https://github.com/user-attachments/assets/0c5e1764-675b-47fe-996e-0f368026cbed" />
  <img width="859" height="719" alt="Evidence 1A" src="https://github.com/user-attachments/assets/80db206b-c7c9-4aa8-9e6c-48037a9a47cf" />
- Successful exploit screenshots (SQLi results, XSS alert, Metasploit shell)
  <img width="1920" height="1080" alt="Evidence 2" src="https://github.com/user-attachments/assets/43559bc2-b205-4bae-b4be-46a81ae9053b" />
  <img width="1920" height="1080" alt="Evidence 2A" src="https://github.com/user-attachments/assets/2c667029-b940-4457-8f63-955404f86ba8" />
  <img width="633" height="720" alt="File upload successful" src="https://github.com/user-attachments/assets/bffde903-c33a-4ed2-a70e-9551bd103b1a" />
  <img width="560" height="720" alt="Evidence 2B" src="https://github.com/user-attachments/assets/0ed63055-08a8-4b73-94c1-4a4ba47a55a5" />
- 403 Forbidden responses after blocking was enabled
  <img width="1920" height="1080" alt="Evidence 3" src="https://github.com/user-attachments/assets/bae16dea-3fe1-40b8-9761-548cb5d93713" />
  <img width="1920" height="1080" alt="Evidence 3A" src="https://github.com/user-attachments/assets/d4f2a115-b8e7-4d80-8ddc-cadb4aab9c92" />
  <img width="491" height="310" alt="Evidence 3C" src="https://github.com/user-attachments/assets/a2beed4e-c9d6-4b4c-9a07-f564b817505d" />

- Kibana confirmation of both the pre-block attacks and the post-block denials
  <img width="1280" height="720" alt="Evidence 4" src="https://github.com/user-attachments/assets/2c6b2042-b714-4693-9d47-e9d1f29e239b" />
  <img width="1280" height="720" alt="Evidence 4A" src="https://github.com/user-attachments/assets/7a63d2bb-65c3-4f41-b70b-bc6ec144fae6" />

## Takeaway
Detective controls and preventive controls are not the same thing, and
confusing them is a common, costly misconfiguration. This project
demonstrates both sides of that distinction with reproducible evidence.
