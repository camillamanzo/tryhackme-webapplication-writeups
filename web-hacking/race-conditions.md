# Race Conditions

## 🔍 Overview

A **race condition** occurs when the behaviour of a program depends on the timing or sequence of events — and that timing can be influenced by an attacker. When multiple threads or processes access and modify a shared resource simultaneously, the outcome becomes unpredictable and exploitable.

> **Web context:** A user submits a request. The server reads a value (e.g. account balance, voucher status, vote count), checks it, and then updates it. If an attacker sends the same request many times simultaneously, multiple reads can happen before any write completes — every request passes the check, and the action is performed multiple times.

---

## ⚙️ Causes

| Cause | Description |
|-------|-------------|
| **Shared resources** | Data or state modified by multiple threads simultaneously — e.g. a balance field updated by concurrent transactions |
| **Parallel execution** | Multiple requests processed at the same time without coordination |
| **Database operations** | Read-check-write sequences that aren't atomic — another process can intervene between the read and the write |
| **Third-party libraries and services** | External dependencies that don't implement their own thread safety |

---

## 🧵 Multi-Threading

Understanding race conditions requires understanding how programs execute concurrently.

### Programs vs Processes vs Threads

| Concept | Description |
|---------|-------------|
| **Program** | Static executable code — instructions stored on disk |
| **Process** | A program in execution — has its own memory space and state |
| **Thread** | Lightweight unit of execution within a process — shares the process's memory and instructions |

### Process States

```
[New] → [Ready] → [Running] → [Waiting] → [Ready] → ...
                      ↓
                 [Terminated]
```

| State | Description |
|-------|-------------|
| **New** | Process just created |
| **Ready** | Loaded into memory, waiting for CPU time |
| **Running** | CPU allocated — instructions executing |
| **Waiting** | Blocked — pending I/O completion or event |
| **Terminated** | Execution complete |

> **Why threads matter for race conditions:** Threads within the same process share memory. If two threads read a value, modify it, and write it back — and those operations interleave — the final state depends entirely on timing. This is the race.

---

## 🏗️ Web Application Architecture

### Client-Server Model

| Role | Description |
|------|-------------|
| **Client** | Program or browser that initiates a request for a service |
| **Server** | Program or system that processes the request and returns a response |

### Multi-Tier Architecture

Modern web applications separate logic into distinct layers — each responsible for a different concern:

| Tier | Role | Technologies |
|------|------|-------------|
| **Presentation** | Renders the UI in the browser | HTML, CSS, JavaScript |
| **Application** | Business logic — processes requests, enforces rules | Node.js, PHP, Django, Rails |
| **Data** | Stores and retrieves application data | MySQL, PostgreSQL, MongoDB |

### Why Architecture Matters for Race Conditions

Data passes through **multiple states** before a change is committed:

```
Request received → Value read from DB → Business logic check → Value updated in DB → Response sent
                         ↑                                              ↑
                   If attacker sends                            Multiple requests
                   simultaneous requests                        pass the check here
                   both reads happen here                       before any write lands
```

> This read-check-write gap is the window a race condition exploits. The wider the window (slower DB, more processing), the easier it is to exploit.

---

## 💥 Exploiting Race Conditions — Burp Suite

Burp Suite's **Repeater** can send parallel requests to exploit timing windows.

### Step-by-Step

```
1. Open Burp Suite → Proxy → Open Browser → Navigate to the target application

2. Identify a sensitive POST request
   (e.g. bank transfer, coupon redemption, vote submission, discount application)
   → Right-click the request → Send to Repeater

3. In the Repeater tab → click '+' → Create a Tab Group
   → Add the request tab to the group

4. Right-click the request tab → Duplicate → repeat 20 times
   (20 parallel requests simulates a realistic race)

5. Choose a send mode and fire:
```

### Send Modes

| Mode | Description | Best For |
|------|-------------|----------|
| **Send group in sequence (single connection)** | All requests sent over one connection before it closes | Baseline testing |
| **Send group in sequence (separate connections)** | Each request uses its own connection, closed between requests | Standard sequential testing |
| **Send group in parallel** | All requests sent simultaneously — maximises timing overlap | Race condition exploitation |

> **Use "Send group in parallel"** to exploit race conditions. All requests hit the server at the same moment, maximising the chance that multiple reads occur before any write completes — causing the action to be performed multiple times.

**Expected outcome:** The application behaves unexpectedly — a balance is deducted multiple times, a one-use coupon is applied repeatedly, a vote is counted more than once, or a rate limit is bypassed.

---

## 🔎 Detection

Race conditions are often protected by controls that assume sequential execution — these controls can be bypassed by timing attacks:

| Control | Bypass Approach |
|---------|----------------|
| **Use once** (coupon, token, invite) | Send parallel requests before the "used" flag is written |
| **Vote once** / **rate once** | Submit simultaneous requests within the check-before-write window |
| **Balance / credit limits** | Send concurrent transfers that each pass the balance check before any deduction lands |
| **Rate limiting (e.g. once per 5 minutes)** | Guess the time window reset and flood requests at that moment |

> **Detection in testing:** Look for endpoints that perform a check *before* an action — discount validation, balance checks, permission verification, submission limits. Any "check then act" pattern with a DB round-trip in between is a candidate.

---

## 🛡️ Mitigation

| Mechanism | Description |
|-----------|-------------|
| **Synchronisation / Locking** | Only one thread can acquire a lock at a time — others are blocked until it is released. Prevents concurrent access to the shared resource |
| **Atomic Operations** | A set of instructions grouped and executed as a single, uninterruptible unit — no other thread can intervene mid-operation |
| **Database Transactions** | Groups multiple DB operations into one unit — either all succeed or all fail together. Prevents partial updates and concurrent modification conflicts |
| **Optimistic Locking** | Read a version number with the data; on write, check the version hasn't changed — if it has, reject the update and retry. Catches concurrent modifications without blocking |
| **Rate Limiting + Idempotency Keys** | Assign a unique key to each intended operation — the server rejects duplicate keys even if the requests arrive simultaneously |

> **The core fix:** Replace any read-check-write pattern with an atomic check-and-set — e.g. a SQL `UPDATE accounts SET balance = balance - 100 WHERE balance >= 100` is atomic. A separate `SELECT` followed by an `UPDATE` is not.
