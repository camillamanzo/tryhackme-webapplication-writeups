# HTTP — HyperText Transfer Protocol

## 🌐 Overview

**HTTP** is the set of rules used to communicate with web servers to request and transmit webpage data. It is the foundation of all data exchange on the web.

**HTTPS** (HTTP Secure) is HTTP with encryption — data is protected using TLS (Transport Layer Security), preventing interception and tampering in transit.

> **HTTP vs HTTPS:** HTTP sends data in plaintext — anyone on the network can read it (credentials, cookies, content). HTTPS encrypts the entire exchange. Always prefer HTTPS; HTTP sessions are trivially interceptable with tools like Wireshark or a MITM proxy.

---

## 🔗 URL — Uniform Resource Locator

A URL is the full instruction set for how to access a resource on the internet. Every component has a specific role.

```
http://user:password@tryhackme.com:80/view-room?id=1#task3
 │         │               │        │     │        │    │
 │         │               │        │     │        │    └── Fragment
 │         │               │        │     │        └─────── Query String
 │         │               │        │     └──────────────── Path
 │         │               │        └────────────────────── Port
 │         │               └─────────────────────────────── Host
 │         └─────────────────────────────────────────────── User Info
 └───────────────────────────────────────────────────────── Scheme
```

| Component | Description | Example |
|-----------|-------------|---------|
| **Scheme** | Protocol to use for the connection | `http`, `https`, `ftp` |
| **User Info** | Optional credentials sent before the host | `user:password` |
| **Host** | Domain name or IP address of the server | `tryhackme.com` |
| **Port** | Port to connect to — omitted if using default (80 for HTTP, 443 for HTTPS) | `80` |
| **Path** | File name or location of the resource on the server | `/view-room` |
| **Query String** | Additional parameters passed to the server — begins with `?`, separated by `&` | `?id=1&sort=asc` |
| **Fragment** | Reference to a location within the returned page — processed client-side, never sent to the server | `#task3` |

> **Security note:** Query strings are visible in the URL and server logs — never pass sensitive data (passwords, tokens) in query strings. Use POST requests with a body instead.

---

## 📨 Requests & Responses

### HTTP Request Structure

```http
GET / HTTP/1.1
Host: tryhackme.com
User-Agent: Mozilla/5.0 Firefox/87.0
Referer: https://tryhackme.com/

```

| Line | Header | Meaning |
|------|--------|---------|
| 1 | **Request line** | HTTP method (`GET`), resource path (`/`), and protocol version (`HTTP/1.1`) |
| 2 | **Host** | The website being requested — required in HTTP/1.1 for virtual hosting |
| 3 | **User-Agent** | Identifies the client software (browser name and version) |
| 4 | **Referer** | The page that linked to this request |
| 5 | *(blank line)* | Marks the end of the headers — required by the HTTP spec |

> **Blank line significance:** The blank line separating headers from the body is mandatory. Without it, the server cannot tell where headers end and body content begins.

---

### HTTP Response Structure

```http
HTTP/1.1 200 OK
Server: nginx/1.15.8
Date: Fri, 09 Apr 2021 12:34:03 GMT
Content-Type: text/html
Content-Length: 98

<html>
<head>
    <title>TryHackMe</title>
</head>
<body>Welcome To TryHackMe.com</body>
</html>
```

| Line | Header | Meaning |
|------|--------|---------|
| 1 | **Status line** | Protocol version and status code — tells the client whether the request succeeded |
| 2 | **Server** | Web server software and version — exposing this is a misconfiguration (information disclosure) |
| 3 | **Date** | Current date, time, and timezone of the web server |
| 4 | **Content-Type** | Type of content being returned — tells the browser how to render it |
| 5 | **Content-Length** | Size of the response body in bytes — client uses this to confirm all data was received |
| 6 | *(blank line)* | Marks the end of the headers |
| 7+ | **Body** | The actual content — HTML, JSON, image data, etc. |

> **Security note:** The `Server` header reveals the web server software and version — this helps attackers identify known CVEs. In hardened deployments, this header is removed or replaced with a generic value.

---

## ⚙️ HTTP Methods

HTTP methods (also called verbs) define the intended action for the request.

| Method | Purpose | Safe? | Idempotent? |
|--------|---------|-------|-------------|
| **GET** | Retrieve a resource — data only, no side effects | ✅ Yes | ✅ Yes |
| **POST** | Submit data to the server — typically creates a resource | ❌ No | ❌ No |
| **PUT** | Submit data to update or replace an existing resource | ❌ No | ✅ Yes |
| **DELETE** | Remove a resource | ❌ No | ✅ Yes |
| **PATCH** | Partially update a resource | ❌ No | ❌ No |
| **HEAD** | Same as GET but returns headers only — no body | ✅ Yes | ✅ Yes |
| **OPTIONS** | Returns the HTTP methods supported by the server for a resource | ✅ Yes | ✅ Yes |

> **Safe** = no server-side state changes. **Idempotent** = calling the same request multiple times produces the same result. GET is both — POST is neither. Attackers exploit method mismatches (e.g. a DELETE endpoint that accepts GET requests).

---

## 📊 HTTP Status Codes

### Ranges

| Range | Category | Meaning |
|-------|----------|---------|
| **1xx** | Informational | First part of the request received — continue sending |
| **2xx** | Success | Request completed successfully |
| **3xx** | Redirection | Client must take further action (follow redirect) |
| **4xx** | Client Error | Something wrong with the request |
| **5xx** | Server Error | Server failed to fulfil a valid request |

### Common Status Codes

| Code | Name | Meaning | Security Relevance |
|------|------|---------|-------------------|
| **200** | OK | Request completed successfully | — |
| **201** | Created | Resource successfully created (e.g. new user, new post) | — |
| **301** | Moved Permanently | Resource has permanently moved — update your bookmarks | Can be abused in redirect chains for phishing |
| **302** | Found | Temporary redirect | Open redirects (`?next=http://evil.com`) are an OWASP risk |
| **400** | Bad Request | Malformed or missing data in the request | — |
| **401** | Unauthorized | Authentication required — not logged in | — |
| **403** | Forbidden | Authenticated but not authorised — lacks permission | Useful in recon — confirms resource exists |
| **404** | Not Found | Resource does not exist | — |
| **405** | Method Not Allowed | HTTP method not accepted for this endpoint | — |
| **500** | Internal Server Error | Server encountered an unhandled error | May reveal stack traces — information disclosure |
| **503** | Service Unavailable | Server overloaded or under maintenance | — |

> **401 vs 403:** 401 means "you need to log in." 403 means "you're logged in but still not allowed." Both confirm the resource exists — useful during enumeration.

---

## 📋 Headers

### Request Headers

Sent by the client with every request to provide context about the client and the request itself.

| Header | Description | Example |
|--------|-------------|---------|
| **Host** | Specifies the website being requested — required for virtual hosting | `Host: tryhackme.com` |
| **User-Agent** | Identifies the browser software and version | `User-Agent: Mozilla/5.0 Firefox/87.0` |
| **Content-Length** | Length of the request body in bytes — used with POST/PUT | `Content-Length: 42` |
| **Accept-Encoding** | Compression methods the browser supports | `Accept-Encoding: gzip, deflate, br` |
| **Cookie** | Sends stored cookies back to the server on each request | `Cookie: session=abc123` |
| **Authorization** | Credentials for protected resources | `Authorization: Bearer <token>` |
| **Referer** | The URL of the page that initiated the request | `Referer: https://tryhackme.com/` |

### Response Headers

Sent by the server with every response to provide metadata about the response and instructions for the client.

| Header | Description | Example |
|--------|-------------|---------|
| **Set-Cookie** | Instructs the browser to store a cookie — sent back on future requests | `Set-Cookie: session=abc123; HttpOnly; Secure` |
| **Cache-Control** | How long the browser should cache the response | `Cache-Control: max-age=3600` |
| **Content-Type** | Type of content being returned — determines how the browser handles it | `Content-Type: text/html; charset=utf-8` |
| **Content-Encoding** | Compression method used on the response body | `Content-Encoding: gzip` |
| **Content-Security-Policy** | Defines allowed sources for scripts, styles, and other resources — prevents XSS | `Content-Security-Policy: default-src 'self'` |
| **X-Frame-Options** | Prevents the page being loaded in an iframe — mitigates clickjacking | `X-Frame-Options: DENY` |
| **Strict-Transport-Security** | Forces HTTPS for future connections | `Strict-Transport-Security: max-age=31536000` |

> **Security headers matter:** `Content-Security-Policy`, `X-Frame-Options`, and `Strict-Transport-Security` are not present by default — they must be explicitly configured. Their absence is a common finding in web application security assessments.

---

## 🍪 Cookies

A **cookie** is a small piece of data stored in the browser, created when a `Set-Cookie` response header is received from the server. Cookies are automatically sent back to the server with every subsequent request to that domain.

### How Cookies Work

```
Client                          Server
  │── GET / ──────────────────────→ │
  │← HTTP/1.1 200 OK ───────────── │
  │  Set-Cookie: session=abc123    │
  │                                │
  │── GET /dashboard ──────────────→│
  │  Cookie: session=abc123        │
  │← HTTP/1.1 200 OK ─────────────→│
```

### Cookie Attributes

| Attribute | Description | Security Impact |
|-----------|-------------|----------------|
| **Name=Value** | The cookie's identifier and stored data | Token or session ID |
| **HttpOnly** | Cookie cannot be accessed by JavaScript | Prevents XSS-based cookie theft |
| **Secure** | Cookie only sent over HTTPS connections | Prevents interception over HTTP |
| **SameSite** | Controls cross-site cookie sending (`Strict`, `Lax`, `None`) | Mitigates CSRF attacks |
| **Expires / Max-Age** | When the cookie expires — session cookies expire when the browser closes | — |
| **Path** | Which URL paths the cookie is sent with | Limits scope |
| **Domain** | Which domains receive the cookie | Limits scope |

### Viewing Cookies

Open **Developer Tools** → **Application** tab → **Cookies** — shows all cookies for the current site, their values, and attributes.

> **Authentication cookies:** The session token stored in a cookie is often the only thing keeping you logged in. Stealing it (via XSS, MITM, or network interception) gives an attacker full account access without needing credentials. This is why `HttpOnly` and `Secure` flags are critical.
