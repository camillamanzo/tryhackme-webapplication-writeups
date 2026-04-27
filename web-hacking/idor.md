# IDOR — Insecure Direct Object Reference

## 🔍 Overview

**IDOR** is a type of **access control vulnerability** that occurs when an application uses a user-controlled value (such as an ID in a URL or request parameter) to directly reference an object — without verifying whether the requesting user is authorised to access it.

```
# Legitimate request — user accesses their own data
https://site.com/user?id=1

# IDOR — user changes the ID to access someone else's data
https://site.com/user?id=2
```

> **IDOR is an authorisation failure, not an authentication failure.** The attacker is logged in — they just shouldn't be able to see this particular resource. The server accepts the request because it never checks whether `id=2` belongs to the requesting user.

---

## 🔢 Types of IDs

IDs aren't always plain integers — applications use various formats to reference objects. Each requires a different approach to test and manipulate.

---

### 🔡 Encoded IDs

Some applications encode IDs before passing them in URLs or request bodies — commonly using **Base64** — to handle binary or special characters safely. Encoding is **not** encryption: it is fully reversible without any key.

| Tool | Purpose | URL |
|------|---------|-----|
| Base64 Decode | Decode an encoded ID to find the underlying value | [base64decode.org](https://www.base64decode.org/) |
| Base64 Encode | Re-encode a modified value to resubmit in the request | [base64encode.org](https://www.base64encode.org/) |

**Workflow:**
```
1. Intercept the request (Burp Suite or DevTools Network tab)
2. Find the encoded parameter — e.g. dXNlcl9pZD0x
3. Decode it → user_id=1
4. Modify the value → user_id=2
5. Re-encode → dXNlcl9pZD0y
6. Resubmit the request with the modified value
```

> **Quick Base64 in terminal:**
> ```bash
> # Decode
> echo "dXNlcl9pZD0x" | base64 -d
>
> # Encode
> echo -n "user_id=2" | base64
> ```

---

### #️⃣ Hashed IDs

Some applications hash the object ID before exposing it — e.g. MD5 of an integer. This appears more secure but can be predictable if the underlying value is simple (low integers hash to well-known values).

**Tool:** [CrackStation](https://crackstation.net/) — reverse-lookup database for common hash:plaintext pairs

**Workflow:**
```
1. Extract the hashed ID from the request
2. Submit to CrackStation — check if it matches a known value (e.g. MD5 of 1, 2, 3...)
3. If matched, hash the next integer and substitute it in the request
```

> MD5 and SHA-1 hashes of small integers (1, 2, 3...) are widely known and trivially reversible via rainbow tables. This is why using a hash of a predictable value is not a security control.

---

### ❓ Unpredictable IDs

When IDs are random (UUIDs, long random tokens) and cannot be guessed or decoded, direct enumeration isn't possible. However, the vulnerability may still exist — the application may not check ownership even if the IDs are hard to guess.

**How to test:**
```
1. Create two separate accounts (Account A and Account B)
2. Log into Account A — note the ID assigned to your resource
3. Log into Account B — attempt to access Account A's resource using its ID
4. If Account B can view Account A's content → IDOR confirmed
```

> This works because IDOR is an authorisation flaw — the server may accept any valid ID regardless of which user it belongs to. Even with unpredictable IDs, the application must still verify ownership server-side.

---

## 📍 Where to Find IDOR

IDOR isn't always visible in the address bar — IDs are passed in many places across an application.

| Location | Description | Example |
|----------|-------------|---------|
| **URL / Address bar** | ID passed directly in the query string or path | `?id=1`, `/users/1/profile` |
| **AJAX requests** | Background API calls not visible in the URL — only visible in DevTools Network tab | `GET /api/user/details?user_id=1` |
| **JavaScript files** | Hardcoded references or dynamically constructed URLs in JS source | `fetch('/api/messages/' + userId)` |
| **POST body parameters** | ID passed in form data or JSON body | `{"account_id": 1, "action": "view"}` |
| **Cookies** | Session or reference data stored in cookies | `user_id=1` in a cookie value |
| **Hidden form fields** | `<input type="hidden" name="user_id" value="1">` in HTML source | Invisible to the user but sent with the form |

---

## 🔬 Worked Example — Finding IDOR via DevTools

A practical workflow for discovering IDOR vulnerabilities on a real application.

```
1. Create an account on the target application
2. Log in

3. Navigate to a page that displays your personal data
   (profile, account settings, order history — anything pre-filled with your info)

4. Open Developer Tools → Network tab

5. Reload the page to capture all requests

6. Look for API calls that load your personal data
   e.g. GET /api/user/details or GET /customers/profile?user_id=1

7. Identify the parameter used to reference your account (user_id, id, account, etc.)

8. Modify the value (increment it, swap it with another user's ID) and resend the request
   → Right-click the request → Copy URL → modify and paste into address bar
   → Or use Burp Suite Repeater to modify and replay the request

9. If another user's data is returned → IDOR vulnerability confirmed
```

> **Tip:** Pay close attention to AJAX requests in the Network tab — the most valuable IDOR findings are often on background API calls that aren't visible in the address bar. These endpoints frequently have weaker access controls than the main pages.

---

## 🛡️ Prevention

| Control | Description |
|---------|-------------|
| **Server-side authorisation checks** | On every request, verify that the authenticated user owns or has permission to access the requested resource — never rely on the client to enforce this |
| **Indirect object references** | Map user-visible IDs to internal IDs server-side — the user never sees or controls the real database ID |
| **Unpredictable IDs** | Use UUIDs or long random tokens instead of sequential integers — harder to enumerate, but not a substitute for proper authorisation checks |
| **Access control testing** | Include IDOR testing in every security assessment — test every endpoint that accepts an object reference |

> **The golden rule:** The presence of a valid ID in a request is never sufficient to grant access. The server must always verify: *does this authenticated user have permission to perform this action on this specific object?*
