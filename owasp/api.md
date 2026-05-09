# OWASP API Security Top 10 — Part 1

## 🔍 Overview

**OWASP** (Open Worldwide Application Security Project) is a non-profit online community that improves application security through freely available principles, documentation, tools, and research.

**APIs** (Application Programming Interfaces) are middleware that facilitates communication between two software components — defined by a set of protocols and definitions. As APIs have become the backbone of modern applications, their unique attack surface has grown significantly.

> **Why APIs need their own Top 10:** API vulnerabilities differ from traditional web app vulnerabilities. APIs expose data and functionality directly — often with less UI-level protection, making issues like broken object-level access or excessive data exposure more impactful and harder to spot.

> **Tool:** [Talent API Tester](https://chrome.google.com/webstore/detail/talend-api-tester-free-ed/aejoelaoggembcahagimdiliamlcdmfm) — browser-based tool for testing and interacting with API endpoints.

---

## 🔴 API1 — Broken Object Level Authorisation (BOLA)

**What it is:** The API equivalent of IDOR. API endpoints receive object identifiers (user IDs, record IDs, account numbers) in requests — and use them to retrieve or manipulate data without verifying whether the requesting user is authorised to access that specific object.

```
# Legitimate — user accesses their own data
GET /api/orders/1001   (authenticated as user who owns order 1001)

# BOLA — user accesses another user's data
GET /api/orders/1002   (same user, different order ID → unauthorised access)
```

**Impact:** Data leakage, exposure of other users' PII, possible full account takeover.

### Mitigation

| Control | Description |
|---------|-------------|
| **User-based authorisation** | Implement access control that validates requests against user policies and role hierarchies — not just authentication |
| **Strict ownership checks** | On every request, verify that the authenticated user owns or has explicit permission to access the requested object |
| **Unpredictable identifiers** | Use strong random values (UUIDs) for object IDs — makes enumeration impractical even if authorisation checks fail |

---

## 🔴 API2 — Broken User Authentication (BUA)

**What it is:** Weaknesses in authentication implementation — absent or improperly validated tokens, invalid session management, or missing authorisation headers. Attackers exploit these to compromise authenticated sessions or bypass authentication entirely.

**Common patterns:**
- No token validation — any value is accepted
- Tokens that never expire or are not invalidated on logout
- Credentials exposed in GET request parameters (visible in logs and browser history)
- No brute force protection — unlimited login attempts

**Impact:** Session hijacking, account takeover, access to sensitive data.

### Mitigation

| Control | Description |
|---------|-------------|
| **Strong passwords** | Enforce high-entropy passwords — minimum length, complexity requirements |
| **Never expose credentials in requests** | Passwords and tokens must not appear in GET parameters, URLs, or response bodies |
| **Secure token implementation** | Use strong, signed JWTs with appropriate expiration — validate on every request |
| **MFA and account lockout** | Require multi-factor authentication where possible; implement account lockout and CAPTCHA to prevent brute force |
| **Hash passwords at rest** | Never store passwords in plaintext — use bcrypt, scrypt, or Argon2 |

---

## 🔴 API3 — Excessive Data Exposure

**What it is:** The API returns more data than the client actually needs — exposing sensitive fields in the response and relying on the front-end to filter them before displaying. If an attacker calls the API directly, they see everything.

```json
// API returns full user object — front-end only displays name
{
  "id": 1,
  "name": "Jane Smith",
  "email": "jane@example.com",
  "account_number": "4929-xxxx-xxxx-1234",   // ← never needed by the client
  "access_token": "eyJhbGc...",               // ← critical secret exposed
  "internal_role": "admin"                    // ← privilege information leaked
}
```

**Impact:** Exposure of PII, account numbers, tokens, internal role information.

### Mitigation

| Control | Description |
|---------|-------------|
| **Filter server-side** | Never delegate sensitive data filtering to the front-end — the API must return only what the client legitimately needs |
| **Regular response audits** | Periodically review API responses to verify only necessary fields are returned |
| **Avoid generic serialisation** | Don't use blanket methods like `to_json()` or `to_string()` that dump entire objects — define explicit response schemas |
| **Automated and manual API testing** | Test all endpoints for data leakage using varied test cases — include both authenticated and unauthenticated requests |

---

## 🔴 API4 — Lack of Resources and Rate Limiting

**What it is:** The API imposes no restrictions on the frequency, volume, or size of requests — allowing clients to consume unlimited resources. This enables brute force attacks, enumeration, and denial of service.

**Examples:**
- Unlimited login attempts → credential brute force
- No limit on file upload size → storage exhaustion
- Unrestricted API polling → server resource exhaustion → DoS
- No throttle on password reset requests → enumeration of valid emails

**Impact:** Server performance degradation, denial of service, successful brute force attacks.

### Mitigation

| Control | Description |
|---------|-------------|
| **Rate limiting** | Enforce a maximum number of API calls per client per time window — return `429 Too Many Requests` when exceeded |
| **Alert on limit breach** | Notify security teams immediately when rate limits are hit — may indicate an attack in progress |
| **Input size limits** | Define maximum values for all parameters — max string length, max array elements, max file size |
| **CAPTCHA** | Protect authentication and sensitive endpoints from automated bot traffic |

---

## 🔴 API5 — Broken Function Level Authorisation (BFLA)

**What it is:** While BOLA targets object-level access (can this user access this specific record?), BFLA targets function-level access (can this user perform this action at all?). Attackers impersonate higher-privilege users or call admin-only API endpoints they shouldn't have access to.

```
# Regular user calls an admin-only endpoint
DELETE /api/admin/users/42   → should be blocked, but isn't

# User escalates role by calling a privilege management endpoint
POST /api/user/setRole  {"role": "admin"}  → should be admin-only
```

**Attacks:** Targets the **authorisation** and **non-repudiation** principles — bypassing function-level checks to perform privileged operations.

**Impact:** Unauthorised data access, data modification, privilege escalation, full admin takeover.

### Mitigation

| Control | Description |
|---------|-------------|
| **Deny by default** | All access denied unless explicitly granted — never assume a user is authorised |
| **Test all authorisation boundaries** | Review every API endpoint against functional-level authorisation requirements — include admin, user, and unauthenticated scenarios |
| **Group and hierarchy enforcement** | Ensure operations are restricted to users belonging to the correct authorised group — account for the application's business logic and role hierarchy |
| **Separate admin APIs** | Keep admin functionality on separate, tightly controlled endpoints rather than exposing it on the same surface as user-facing endpoints |
