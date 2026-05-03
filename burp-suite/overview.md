# Burp Suite Basics

## 🔍 Overview

**Burp Suite** is a Java-based framework designed for web application penetration testing. It acts as a proxy between the browser and the target server — capturing, inspecting, and manipulating HTTP/HTTPS traffic in real time.

| Edition | Description |
|---------|-------------|
| **Community** | Free — core tools available with some rate limitations |
| **Professional** | Full-featured — automated vulnerability scanner, unlimited fuzzer/brute-forcer, project saving |
| **Enterprise** | Designed for continuous automated scanning at scale (CI/CD integration) |

---

## 🧰 Features

| Module | Description |
|--------|-------------|
| **Proxy** | Intercepts and modifies HTTP/HTTPS requests and responses between browser and server |
| **Repeater** | Captures, modifies, and resends requests manually — ideal for testing and refining payloads |
| **Intruder** | Automates spraying endpoints with requests — brute force, fuzzing, and parameter testing |
| **Decoder** | Transforms data — decode captured values or encode payloads before sending |
| **Comparer** | Compares two pieces of data at word or byte level — useful for spotting subtle differences in responses |
| **Sequencer** | Analyses the randomness of tokens (e.g. session cookies) to identify weak entropy |
| **Extensions** | Extend Burp's functionality with custom modules — written in Java, Python, or Ruby |
| **BApp Store** | Marketplace for third-party Burp extensions — accessed via the Extender module |

---

## 🔧 Installation

| Platform | Method |
|----------|--------|
| **Kali Linux** | Pre-installed — if missing, install via `sudo apt install burpsuite` |
| **Linux / macOS / Windows** | Download installer from [portswigger.net/burp/releases](https://portswigger.net/burp/releases) |

---

## 🖥️ Dashboard

The dashboard is divided into four quadrants (clockwise from top-left):

| Quadrant | Description |
|----------|-------------|
| **Tasks** | Define and manage background tasks — default task is *Live Passive Crawl*, which logs all visited pages |
| **Event Log** | Records Burp Suite actions — proxy start, intercepted requests, errors |
| **Issue Activity** | *(Pro only)* Vulnerabilities identified by the automated scanner — ranked by severity, filterable by certainty |
| **Advisory** | Detailed info on identified vulnerabilities — includes descriptions, references, and remediation guidance. Exportable as a report |

> Click any **?** icon to open a help window specific to that section.

---

## 🧭 Navigation

| Feature | Description |
|---------|-------------|
| **Module selection** | Top menu bar — click any module to switch to it |
| **Sub-tabs** | Each module has sub-tabs accessible via the second menu bar below the main one |
| **Detaching tabs** | View modules in separate windows via *Window → Detach* |

### Keyboard Shortcuts

| Shortcut | Module |
|----------|--------|
| `Ctrl+Shift+D` | Dashboard |
| `Ctrl+Shift+T` | Target |
| `Ctrl+Shift+P` | Proxy |
| `Ctrl+Shift+I` | Intruder |
| `Ctrl+Shift+R` | Repeater |

---

## ⚙️ Options / Settings

| Setting Type | Scope |
|-------------|-------|
| **Global (User) Settings** | Apply to the entire Burp Suite installation — persist across sessions |
| **Project Settings** | Apply only to the current session — lost on close in Community edition |

> Burp Suite Community does not support saving projects — all project settings are lost when the application is closed.

**Settings window features:**
- **Search** — find specific settings by keyword
- **Type filter** — toggle between User (global) and Project settings
- **Categories** — browse settings by category

---

## 🔀 Burp Proxy

The Proxy is Burp Suite's core feature — it sits between your browser and the target server, giving you complete control over all traffic.

| Feature | Description |
|---------|-------------|
| **Request interception** | Requests appear in the Proxy tab — can be forwarded, dropped, edited, or sent to other modules. Toggle with the *Intercept is On/Off* button |
| **Capture and logging** | All requests are logged in HTTP History even when interception is off |
| **WebSocket support** | Captures and logs WebSocket communications alongside HTTP |
| **Response interception** | Off by default — enable via *Intercept responses based on the following rules* checkbox |
| **Match and Replace** | Uses regex to automatically modify requests or responses as they pass through |

---

### 🦊 Connecting via FoxyProxy (Firefox)

FoxyProxy routes Firefox traffic through Burp's proxy listener.

```
1. Install FoxyProxy
   → https://addons.mozilla.org/en-US/firefox/addon/foxyproxy-basic/

2. Click the FoxyProxy icon → Options → Add

3. Configure proxy:
   Title:    Burp
   IP:       127.0.0.1
   Port:     8080

4. Save the config

5. Enable interception in Burp Suite:
   Proxy tab → Intercept is On

6. Activate in Firefox:
   Click FoxyProxy icon → select your "Burp" config

7. Test: open any website — the browser should hang and the request
   should appear in Burp's Proxy tab
```

> When the proxy config is active, every browser request hangs until you forward or drop it in Burp.

**Right-click options on an intercepted request:**
- Forward — send to the server
- Drop — discard the request
- Send to Repeater / Intruder / other modules
- Copy to clipboard

---

### 🔒 Fixing HTTPS / TLS Certificate Errors

By default, Burp uses its own CA certificate — browsers flag HTTPS sites as untrusted. Import Burp's CA cert to fix this.

```
1. Download the CA certificate:
   Navigate to http://burp/cert in Firefox → save the .der file

2. Open Firefox certificate settings:
   Address bar → about:preferences → search "Certificates" → View Certificates

3. Import the downloaded .der file

4. Set trust:
   Check "Trust this CA to identify websites"
```

> After importing, Burp's CA is trusted by Firefox — TLS-enabled sites are fully accessible through the proxy without certificate warnings.

---

## 🎯 Scope & Site Map

### Site Map

The Target tab's **Site Map** builds a visual tree of the target application — pages, endpoints, parameters, and APIs discovered during testing.

- Generate a map by browsing the application through the proxy
- Use it to identify endpoints for further testing
- Export the site map for documentation

### Issue Definitions

A built-in reference list of web vulnerabilities — each entry includes a description, severity, and remediation guidance. Useful for writing reports when vulnerabilities are found.

### Scope Settings

Scope controls which traffic Burp captures and processes — prevents logging irrelevant requests from other sites.

```
1. Target tab → right-click the target in the site map list → Add to Scope

2. Restrict proxy to in-scope traffic only:
   Proxy Settings → "And URL Is in Target Scope" option
```

---

## 🧪 Practical — XSS Attack via Proxy

A step-by-step demonstration of using Burp Proxy to intercept and modify a request to deliver an XSS payload.

```
1. Activate Burp Proxy intercept:
   Proxy tab → Intercept is On

2. In Firefox, fill out and submit a form on the target site

3. The request appears in Burp's Proxy tab — intercepted before it reaches the server

4. In the request body, locate the target parameter and replace its value
   with an XSS payload:
   <script>alert("Successful XSS")</script>

5. URL-encode the payload to ensure safe transmission:
   Select the payload text → Ctrl+U

6. Forward the request

7. If the application is vulnerable, a popup alert fires in the browser
```

> **Why URL-encode?** Special characters like `<`, `>`, and `"` have meaning in HTTP — encoding them (`%3C`, `%3E`, `%22`) ensures they are transmitted correctly and decoded by the server before being written into the page.
