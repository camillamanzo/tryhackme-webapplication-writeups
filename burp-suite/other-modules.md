# Burp Suite — Other Modules

## 🔣 Decoder

### Overview

**Decoder** transforms data between various encoding formats — useful for decoding captured values, encoding payloads before sending, and generating hash checksums. It mirrors CyberChef's **Magic** operation with its Smart Decode feature.

---

### Encoding / Decoding Formats

| Format | Description | Example |
|--------|-------------|---------|
| **Plain** | Raw, untransformed text | `hello world` |
| **URL** | Encodes special characters for safe URL transmission — replaces chars with `%xx` hex values | `/` (ASCII 47 → hex 2F) becomes `%2F` |
| **HTML** | Replaces special characters with HTML entities — prevents XSS and ensures safe rendering | `&quot;` → `"` |
| **Base64** | Converts binary or text data into ASCII-compatible format | `admin` → `YWRtaW4=` |
| **ASCII Hex** | Converts text to its hexadecimal ASCII representation | `ASCII` → `4153434949` |
| **Hex / Octal / Binary** | Numeric format conversions — integer input only | `255` → `FF` (hex) |
| **Gzip** | Compresses data — reduces size for faster transfer | Binary compressed output |
| **Hex Format** | View and edit raw data in hexadecimal format | Useful for binary analysis |
| **Smart Decode** | Recursively attempts to auto-detect and decode the encoding — works like CyberChef's Magic | Handles nested encodings automatically |

---

### Hashing

Decoder can generate hash checksums of any input using multiple algorithms:

- MD5, SHA-1, SHA-256, SHA-512 (and others)
- Useful for verifying file integrity or comparing hash values during testing

> **Tip:** Use Decoder to quickly check whether a cookie or token value is Base64-encoded, URL-encoded, or hashed — paste the value and click Smart Decode to let Burp attempt identification automatically.

---

## ⚖️ Comparer

### Overview

**Comparer** performs a side-by-side diff of two pieces of data — at the character (ASCII) or byte level. It highlights differences between two requests or responses, making subtle changes immediately visible.

> **When to use it:** Comparing a failed vs successful login response to find which field changes, identifying differences between two user accounts' responses, or spotting what changes after a privilege escalation attempt.

### Example — Comparing Login Responses

```
1. Capture a login request with wrong credentials in Proxy
2. Send the request to Repeater (Ctrl+R)
3. Send the response to Comparer:
   right-click response → Send to Comparer

4. In Repeater, modify the request with correct credentials and send
5. Send this second response to Comparer

6. In Comparer, select both entries → Compare (Words or Bytes)
   → Differences are highlighted — reveals exactly what changed
   between a failed and successful authentication response
```

---

## 🎲 Sequencer

### Overview

**Sequencer** evaluates the **randomness** of tokens — session cookies, CSRF tokens, password reset links, and any other value that should be unpredictable. Weak entropy means tokens can be predicted, enabling session hijacking or token forgery.

| Mode | Description |
|------|-------------|
| **Live Capture** | Forwards a request to the server thousands of times — collects and analyses the tokens generated in each response |
| **Manual Load** | Analyse a pre-collected set of tokens — no requests sent to the target |

### Example — Analysing Session Token Entropy

```
1. Capture a login request in Proxy → Send to Sequencer

2. In the "Token Location Within Response" section:
   → Select the token type — e.g. Form Field for loginToken,
     or Cookie for a session cookie

3. Click Start Live Capture
   → Burp sends the request repeatedly, collecting token samples

4. At ~10,000 samples → click Analyse Now
   (or enable Auto Analyse to run automatically every 2,000 requests)
```

### Result Analysis

| Field | Description |
|-------|-------------|
| **Overall Result** | Summary assessment of token security — pass/fail with confidence level |
| **Effective Entropy** | Measured randomness in bits — higher is better. Below 32 bits is considered weak |
| **Reliability** | Confidence in the accuracy of the entropy estimate — increases with sample size |
| **Sample** | Details about the collected tokens — count, length distribution, and characteristics |

> **What to look for:** Low effective entropy (< 64 bits) suggests tokens are predictable and may be forgeable. Patterns in the token structure — sequential numbers, timestamps, or repeating segments — are immediate red flags.

---

## 🗂️ Organiser

### Overview

**Organiser** stores annotated, read-only copies of HTTP requests for later reference — a persistent notepad for interesting requests found during testing.

| Feature | Description |
|---------|-------------|
| **Read-only copy** | Requests are stored exactly as captured — cannot accidentally modify the original |
| **Annotations** | Add notes to each saved request — document why it was flagged, what was found, or what to test next |
| **Table view** | All saved requests displayed in a sortable table — easy to browse and retrieve |

> **When to use it:** Save interesting requests as you find them during a test — login endpoints, admin API calls, password reset flows — so you can return to them later without losing context or having to rediscover them in the Proxy history.
