# Walking an Application

## 🌐 Overview

Before using automated tools, a good web app tester manually walks through an application using built-in browser features — no special software needed. The goal is to map the attack surface, understand how the app works, and spot vulnerabilities that scanners miss.

> **Why manual review first?** Automated scanners find known patterns. Manual review finds logic flaws, exposed comments, hidden endpoints, and misconfigurations that are invisible to tools. Page source and DevTools are the starting point for every web app assessment.

---

## 📄 Page Source

The **page source** is the raw HTML, CSS, and JavaScript returned by the web server in response to a request. It is exactly what the browser receives before it parses and renders it into the visual page you see.

> **Page source vs Inspector:** Page source shows the original response from the server — static and unmodified. The Inspector (DevTools) shows the live DOM — which may have been modified by JavaScript after the page loaded. Both are useful; they show different things.

### What to Look For

| Finding | Why It Matters |
|---------|---------------|
| **HTML comments** (`<!-- ... -->`) | Developers leave notes, credentials, TODO items, and disabled code in comments — visible in source but not on the rendered page |
| **Hidden form fields** | `<input type="hidden">` fields pass data the user can't see — but can modify |
| **Hardcoded credentials or API keys** | Sometimes left in JS files or HTML by accident |
| **Links to internal pages or admin panels** | Paths referenced in source but not linked in the UI |
| **JavaScript file references** | External `.js` files often contain application logic, API endpoints, and secrets |
| **Framework or version indicators** | Comments, meta tags, or file names that reveal which framework/version is in use — helps identify CVEs |
| **Inline JavaScript** | Business logic, validation rules, or API calls written directly in the HTML |

### How to View Page Source

| Method | How |
|--------|-----|
| **Right-click** | Right-click anywhere on the page → *View Page Source* |
| **URL prefix** | Add `view-source:` before the URL — e.g. `view-source:https://tryhackme.com` |
| **Browser menu** | Browser menu → Developer Tools or View Source (location varies by browser) |

---

## 🛠️ Developer Tools

**Developer Tools** (DevTools) is the built-in browser toolkit for inspecting, debugging, and interacting with web applications in real time. Available in all major browsers — open with `F12` or `Ctrl+Shift+I` (`Cmd+Option+I` on Mac).

> DevTools is one of the most powerful tools in a web tester's arsenal — and it requires no installation. Every browser has it.

---

### 🔍 Inspector

The Inspector shows a **live, editable representation** of the current page's DOM (Document Object Model) — the browser's internal structure of the page after all HTML has been parsed and JavaScript has run.

**Key uses:**

| Action | How | Why Useful |
|--------|-----|-----------|
| **View live DOM** | Click any element to highlight it on the page | Understand page structure; find hidden elements |
| **Edit HTML in real time** | Double-click any element in the Inspector | Modify content, reveal hidden sections, bypass client-side restrictions |
| **Change CSS properties** | Click any style in the Styles panel | Remove `display: none` to reveal hidden UI elements (e.g. premium-only blocks, admin panels) |
| **Find element by clicking** | Inspector cursor icon → click on page element | Jump directly to the source of any visible element |

> **Common attack use:** A `display: none` CSS property hides elements visually — but they still exist in the DOM. Changing it to `display: block` in the Inspector can reveal hidden content, disabled buttons, or locked features. This is a client-side bypass — the server may still enforce the restriction, but it's worth checking.

---

### 🐛 Debugger

The Debugger is intended for **debugging JavaScript** — stepping through code execution, inspecting variables, and setting breakpoints.

| Browser | Name |
|---------|------|
| Chrome / Edge | Sources |
| Firefox | Debugger |
| Safari | Debugger |

**Key features:**

| Feature | Description |
|---------|-------------|
| **Pretty print** (`{}`) | Formats minified JavaScript into readable, indented code — essential since production JS is almost always minified onto a single line |
| **Breakpoints** | Click a line number to pause JS execution at that point — lets you inspect variable values and step through logic one line at a time |
| **Watch expressions** | Monitor the value of specific variables as execution progresses |
| **Call stack** | Shows the chain of function calls that led to the current point — useful for tracing how a function was reached |
| **Scope panel** | Shows all variables in scope at the current breakpoint — local, closure, and global |

> **Security use:** The Debugger lets you step through authentication logic, form validation, and token handling in real time. You can find where values are checked, modify them before the check runs, or identify endpoints being called that aren't visible elsewhere.

---

### 🌐 Network

The Network tab **records every HTTP request and response** the browser makes — including requests that happen in the background without any visible page change.

**Key uses:**

| Use | Description |
|-----|-------------|
| **Monitor all requests** | See every resource the page loads — HTML, CSS, JS, images, fonts, API calls |
| **Inspect request/response headers** | View the full HTTP headers for any request — cookies, authorisation tokens, content types |
| **View request/response bodies** | See exactly what data is sent to and received from the server |
| **Identify hidden API endpoints** | Background requests reveal endpoints not linked in the UI |
| **Replay and modify requests** | Right-click a request → copy as `curl` or send to Burp Suite for further manipulation |

### AJAX

**AJAX** (Asynchronous JavaScript and XML) is the technique used to send and receive data from a server in the background — without reloading or visibly changing the current page. Most modern web applications rely on it heavily.

```
Traditional page load:    User action → full page reload → new HTML from server
AJAX request:             User action → background request → partial page update
```

> **Why AJAX matters for testing:** AJAX calls are invisible on the surface — the page doesn't visibly change. The Network tab is the only way to see them. These background requests often carry sensitive data, hit internal API endpoints, and use authentication tokens that can be extracted and replayed. Always check the Network tab for AJAX traffic before concluding there is nothing to test.
