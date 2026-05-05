# Burp Suite — Intruder

## 🔍 Overview

**Intruder** automates request manipulation — fuzzing parameters, brute-forcing login forms, and testing endpoints with large sets of payloads. It takes a captured request, lets you define injection positions, and fires payloads into those positions repeatedly and automatically.

> **Community edition limitation:** Intruder in the free Community edition is rate-limited (throttled). For unrestricted speed, Burp Suite Professional is required. Tools like **ffuf** or **Hydra** are commonly used as free alternatives for brute-forcing.

---

## 🖥️ Interface

| Tab | Description |
|-----|-------------|
| **Positions** | Define the attack type and mark where payloads will be injected in the request |
| **Payloads** | Configure the values to insert — wordlists, numbers, custom lists, processing rules |
| **Resource Pool** | Allocate resources across concurrent Intruder tasks |
| **Settings** | Configure attack behaviour — flag responses containing specific text, handle redirects |

> **Target** is automatically populated when a request is forwarded from the Proxy.

---

## 📍 Positions

Positions define where in the request payloads will be substituted. Marked with `§` delimiters.

| Control | Description |
|---------|-------------|
| `§` markers | Wrap the value to be replaced — e.g. `username=§admin§` |
| **Add §** | Highlight text in the request editor → click Add § to mark it as a position |
| **Clear §** | Remove all defined positions — blank canvas to define your own from scratch |
| **Auto §** | Automatically detect likely injection positions based on the request structure |

---

## 🎯 Payloads

### Payload Sets

Each position can have its own payload set. When multiple positions exist, sets are assigned top-to-bottom, left-to-right:

```
username=§pentester§&password=§Expl01ted§
         ↑ Payload Set 1              ↑ Payload Set 2
```

### Payload Settings

Options vary by payload type. For **Simple List**:
- Manually add or remove individual payload values
- Load from a file (wordlist)
- Paste a list directly

### Payload Processing

Apply transformation rules to each payload before it is sent:

| Rule Type | Example |
|-----------|---------|
| Capitalise first letter | `admin` → `Admin` |
| Add prefix/suffix | `admin` → `admin123` |
| Skip if matches regex | Skip payloads containing special characters |
| Hash | MD5/SHA1 the payload before sending |

### Payload Encoding

URL-encoding is applied by default — customise or disable per payload set as needed.

---

## ⚔️ Attack Types

| Type | Behaviour | Best For |
|------|-----------|----------|
| **Sniper** | Cycles through payloads one at a time — inserts each into every defined position sequentially | Single-position attacks — brute force, fuzzing one parameter |
| **Battering Ram** | Same payload inserted into **all** positions simultaneously | Testing identical values across multiple fields at once |
| **Pitchfork** | Multiple payload sets iterated in parallel — first item from list 1 + first item from list 2, then second + second, etc. | Known username:password pairs — credential stuffing with matched lists |
| **Cluster Bomb** | Tests every combination of all payload sets — cartesian product | Unknown pairings — e.g. 3 usernames × 3 passwords = 9 total requests |

> **Total requests — Sniper:** `number of payloads × number of positions`
> **Total requests — Cluster Bomb:** `payloads in set 1 × payloads in set 2 × ...`

### Attack Type Comparison

```
Sniper:         pos1=payload1   pos2=original
                pos1=original   pos2=payload1
                pos1=payload2   pos2=original
                ...

Battering Ram:  pos1=payload1   pos2=payload1
                pos1=payload2   pos2=payload2
                ...

Pitchfork:      pos1=list1[0]   pos2=list2[0]
                pos1=list1[1]   pos2=list2[1]
                ...

Cluster Bomb:   pos1=list1[0]   pos2=list2[0]
                pos1=list1[0]   pos2=list2[1]
                pos1=list1[1]   pos2=list2[0]
                ...
```

---

## 💥 Attack Example — Credential Brute Force

### Setup

```bash
# Download credential wordlist
wget http://10.129.180.94:9999/Credentials/BastionHostingCreds.zip
```

```
1. Activate FoxyProxy in Firefox
2. Navigate to the target login page and attempt a login
3. Intercept the POST request in Proxy → Ctrl+I to send to Intruder
4. In Positions tab → mark username and password values with § §
5. In Payloads tab → load the wordlist file for the appropriate position
6. Start Attack
7. Sort results by Length column — shorter response = successful login
   (login page reloads on failure, dashboard loads on success → different sizes)
```

---

## 🔧 Building a Macro — Handling CSRF Tokens

Some applications use **CSRF tokens** or **login tokens** that change with every request. Intruder will fail if it reuses a stale token. A **macro** automates token refresh before each attack request.

### Step 1 — Create the Macro

```
Settings → Sessions → Macros → Add
→ Select the GET /admin/login/ request from the list
→ Confirm → name and save the macro
```

### Step 2 — Create a Session Handling Rule

```
Settings → Sessions → Session Handling Rules → Add

Scope tab:
  → Tools Scope → select Intruder only
  → URL Scope → add target URL to custom scope

Details tab:
  → Add → Run a Macro → select your macro
```

> This macro fires before every Intruder request — fetching a fresh token and injecting it automatically.

### Step 3 — Restrict Which Parameters Are Updated

```
Session Handling Rules → Edit

→ "Update only the following parameters and headers"
   → type: loginToken → Add → Close

→ "Update only the following cookies"
   → type: session → Add → Close

→ OK
```

### Step 4 — Run the Attack

```
Start Attack

Expected: 302 Redirect responses for every request
If 403 errors appear → macro is not working — verify scope and parameter settings

Sort by Length → the successful login response will be significantly
shorter than the failed ones (redirect target page vs. error page)
```

> After identifying valid credentials, refresh the login page before submitting them — the old session/token may have expired during the attack.
