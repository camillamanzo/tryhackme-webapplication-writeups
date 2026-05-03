# 🌐 TryHackMe Write-Ups

Web security write-ups from TryHackMe labs covering web fundamentals, application hacking, and offensive web tooling. I have summarised the topics I have studied so far, and will continue to do so for the whole Web Fundamentals learning path.

> **About me:** CompTIA Security+ certified professional with 4+ years in full-stack development and IT/OT consulting, now deepening hands-on cybersecurity skills. Background in RBAC implementation, MES infrastructure, and secure application development.

[![TryHackMe](https://img.shields.io/badge/TryHackMe-camillamanzo-red?style=flat&logo=tryhackme)](https://tryhackme.com/p/CamiM98)
[![Security+](https://img.shields.io/badge/CompTIA-Security%2B-red?style=flat)](https://www.credly.com/badges/0ab21ebf-70c0-4396-bbf4-76f8c4489d30/public_url)
[![LinkedIn](https://img.shields.io/badge/LinkedIn-Camilla%20Manzo-blue?style=flat&logo=linkedin)](https://www.linkedin.com/in/camilla-manzo-a18a65228/)

---

## 📂 Write-Up Index

### 🌍 Web Fundamentals
| Room | Difficulty | Topics | Write-Up |
|------|-----------|--------|----------|
| DNS in Detail | Easy | domain hierarchy, record types, DNS resolution process | [📄 Read](web-fundamentals/dns.md) |
| HTTP in Detail | Easy | URLs, request/response structure, methods, status codes, headers, cookies | [📄 Read](web-fundamentals/http.md) |
| Websites | Easy | HTML structure, JavaScript, HTML injection vs XSS | [📄 Read](web-fundamentals/websites.md) |

### 🌍 Web Hacking
| Room | Difficulty | Topics | Write-Up |
|------|-----------|--------|----------|
| Walking An Application | Easy | page source analysis, inspector, debugger, network | [📄 Read](web-hacking/app.md) |
| Content Discovery | Easy | manual discovery, OSINT, automated discovery | [📄 Read](web-hacking/content-discovery.md) |
| Subdomain Enumeration | Easy | OSINT, DNS brute force, virtual host fuzzing | [📄 Read](web-hacking/subdom-enum.md) |
| Authentication Bypass | Easy | username enumeration, brute force, logic flaws, cookie tampering | [📄 Read](web-hacking/auth-bypass.md) |
| IDOR | Easy | encoded/hashed/unpredictable IDs, IDOR locations, exploitation workflow | [📄 Read](web-hacking/idor.md) |
| File Inclusion | Medium | path traversal, LFI scenarios & bypasses, RFI, remediation | [📄 Read](web-hacking/file-inclusion.md) |
| SSRF | Medium | regular vs blind SSRF, attack scenarios, finding SSRF, bypassing deny/allow lists, open redirect | [📄 Read](web-hacking/ssrf.md) |
| XSS | Medium | reflected, stored, DOM-based & blind XSS, payloads, context escaping, polyglots | [📄 Read](web-hacking/xss.md) |
| Race Conditions | Medium | multi-threading, web app architecture, exploitation with Burp, detection & mitigation | [📄 Read](web-hacking/race-conditions.md) |
| Command Injection | Medium | blind vs verbose injection, shell operators, Linux/Windows payloads, remediation | [📄 Read](web-hacking/command-injection.md) |
| SQL & SQL Injection | Medium | SQL fundamentals, in-band/blind/out-of-band SQLi, union-based exploitation, remediation | [📄 Read](web-hacking/sqli.md) |

### 🌍 Burp Suite
| Room | Difficulty | Topics | Write-Up |
|------|-----------|--------|----------|
| Burp Suite Overview | Easy | proxy setup, FoxyProxy, TLS certificates, site map, scope, XSS via proxy | [📄 Read](burp-suite/overview.md) |
| Burp Suite Repeater | Easy | interface, message analysis, Inspector, SQLi exploitation walkthrough | [📄 Read](burp-suite/repeater.md) |
---

## 🛠️ Tools Used

![Burp Suite](https://img.shields.io/badge/-Burp%20Suite-orange?style=flat)
![SQLMap](https://img.shields.io/badge/-SQLMap-red?style=flat)
![Ffuf](https://img.shields.io/badge/-Ffuf-black?style=flat)
![Nmap](https://img.shields.io/badge/-Nmap-black?style=flat)
![Wireshark](https://img.shields.io/badge/-Wireshark-blue?style=flat)
![curl](https://img.shields.io/badge/-curl-grey?style=flat)

---

## 📝 Write-Up Format

Each write-up follows this structure:

```
## Room: [Name]
**Difficulty:** Easy / Medium / Hard
**Category:** Web Fundamentals / Web Hacking / OWASP / Offensive Tooling / CTF

### Objective
What the room is about and what I aimed to learn.

### Approach & Methodology
Step-by-step walkthrough of my process — tools used, commands run, reasoning.

### Key Findings / Flags
Redacted or summarised — no full spoilers.

### What I Learned
Takeaways, new techniques, and how they connect to real-world scenarios.

### Tools & Commands Used
Quick reference of commands and flags used in the room.
```

---

## 🎯 Learning Path Progress

- [ ] Web Security Essentials
- [ ] Web Application Basics
- [ ] JavaScript Essentials
- [ ] SQL Fundamentals
- [ ] Burp Suite Basics
- [ ] Burp Suite Repeater
- [ ] SQL Injection
- [ ] Cross-Site Scripting (XSS)
- [ ] Command Injection
- [ ] Upload Vulnerabilities
- [ ] OWASP Top 10 — IAAA Failures
- [ ] OWASP Top 10 — Application Design Overflow
- [ ] OWASP Top 10 — Insecure Data Handling
- [ ] SQLMap
- [ ] Ffuf
- [ ] Pickle Rick

---

## 📬 Contact

📧 camillamanzo98@gmail.com
🔗 [LinkedIn](https://www.linkedin.com/in/camilla-manzo-a18a65228/)
💻 [GitHub](https://github.com/camillamanzo)

---

*Write-ups are my own analysis and methodology notes. No full flag spoilers — the goal is to document thinking, not shortcut others' learning.*
