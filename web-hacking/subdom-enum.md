# Subdomain Enumeration

## 🔍 Overview

**Subdomain enumeration** is the process of finding valid subdomains for a target domain — expanding the attack surface beyond the main site. Subdomains often host forgotten services, development environments, admin panels, or legacy applications with weaker security than the primary domain.

> **Why it matters:** `admin.target.com`, `dev.target.com`, `staging.target.com` — these are common subdomains that may run unpatched software, expose internal tools, or lack authentication entirely. They would never be found by testing `www.target.com` alone.

---

## 🌍 OSINT

Passive subdomain discovery using publicly available information — no direct interaction with the target server.

---

### 🔒 SSL/TLS Certificates — Certificate Transparency Logs

When a **Certificate Authority (CA)** issues an SSL/TLS certificate for a domain, it is publicly logged in **Certificate Transparency (CT) logs** — a requirement designed to prevent misissued certificates. These logs are a goldmine for subdomain enumeration.

**How to use:**

→ [crt.sh](https://crt.sh/) — searchable database of current and historical certificates

```
# Search for all certificates issued for a domain
https://crt.sh/?q=target.com

# Use % as a wildcard to find subdomains
https://crt.sh/?q=%25.target.com
```

> **Tip:** Try `http://` instead of `https://` if crt.sh doesn't load — the site occasionally has TLS issues. Historical results are just as useful as current ones — a subdomain may have had a certificate in the past even if it no longer does.

---

### 🔎 Search Engines

Search engines index subdomains that have been crawled — using the `site:` operator with a wildcard surfaces them without touching the target.

```
# Find all indexed subdomains of target.com except www
site:*.target.com -site:www.target.com
```

> This returns only results under `target.com` while excluding the main `www` subdomain — leaving only subdomains in the results. Combine with `inurl:`, `intitle:`, or `filetype:` for more targeted results.

---

### 🐍 Sublist3r

**Sublist3r** automates passive subdomain discovery by querying multiple public sources simultaneously — search engines, DNS datasets, certificate logs, and more.

```bash
# Basic usage — discover subdomains for a domain
./sublist3r.py -d acmeitsupport.thm
```

| Flag | Description |
|------|-------------|
| `-d` | Target domain |
| `-t` | Number of threads — e.g. `-t 50` |
| `-o` | Save results to a file — e.g. `-o subdomains.txt` |
| `-b` | Enable brute-force mode alongside passive discovery |
| `-e` | Specify which engines to use — e.g. `-e google,dnsdumpster` |

---

## 💥 Brute Force

Active discovery — directly querying DNS or the web server with large lists of potential subdomain names.

---

### 🗂️ DNS Brute Force

Tests a wordlist of commonly used subdomain names against the target domain — any that resolve to a valid IP exist.

**Tool: dnsrecon**

```bash
# -t brt: brute-force mode
# -d: target domain
dnsrecon -t brt -d acmeitsupport.thm
```

| Flag | Description |
|------|-------------|
| `-t brt` | Brute-force mode — tests each entry in the wordlist as a subdomain |
| `-d` | Target domain |
| `-D` | Custom wordlist — e.g. `-D /usr/share/wordlists/subdomains.txt` |

> Brute-force DNS enumeration generates real DNS queries — it is active reconnaissance and may appear in the target's DNS server logs.

---

## 🖥️ Virtual Hosts

Some subdomains are never publicly registered or indexed — they exist only on a **private DNS server** or in a developer's local `hosts` file. Standard OSINT and DNS brute-forcing won't find them.

### How Virtual Hosting Works

A single web server can host multiple websites simultaneously. When a request arrives, the server reads the **`Host` header** to determine which site to serve.

```
Request to 10.81.191.148 with Host: www.acmeitsupport.thm     → serves main site
Request to 10.81.191.148 with Host: admin.acmeitsupport.thm   → serves admin panel
```

The IP address is the same — only the `Host` header differs.

### Hosts File

The `hosts` file maps domain names to IP addresses locally — before DNS is consulted. Developers use it to access internal subdomains that aren't in public DNS.

| OS | Location |
|----|----------|
| Linux / macOS | `/etc/hosts` |
| Windows | `C:\Windows\System32\drivers\etc\hosts` |

---

### 🔧 Virtual Host Fuzzing with ffuf

By fuzzing the `Host` header, you can discover subdomains that only exist on the server — not in public DNS.

**Basic command:**
```bash
ffuf -w /usr/share/wordlists/SecLists/Discovery/DNS/namelist.txt \
     -H "Host: FUZZ.acmeitsupport.thm" \
     -u http://10.81.191.148
```

| Flag | Description |
|------|-------------|
| `-w` | Wordlist path |
| `-H` | Add or override a header — `FUZZ` is replaced with each wordlist entry |
| `-u` | Target URL — the server IP (not the domain, since we're fuzzing the Host header) |

**The problem:** Every request gets a response — even invalid subdomains return something (usually the default site). The output is flooded with false positives.

**Solution — filter by response size:**
```bash
ffuf -w /usr/share/wordlists/SecLists/Discovery/DNS/namelist.txt \
     -H "Host: FUZZ.acmeitsupport.thm" \
     -u http://10.81.191.148 \
     -fs {size}
```

| Flag | Description |
|------|-------------|
| `-fs` | Filter out responses of a specific size in bytes — run without it first, note the most common response size (the default site), then re-run filtering that size out |

**Workflow:**
```
1. Run without -fs → note the most common response size (default page)
2. Re-run with -fs {that size} → only non-default responses remain
3. Remaining results = potential valid virtual hosts
```

> **Why target the IP and not the domain?** When fuzzing virtual hosts, you point ffuf at the server's IP address and manipulate the `Host` header manually. If you used the domain, your modified `Host` header would conflict with DNS resolution.
