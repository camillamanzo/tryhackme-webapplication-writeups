# Content Discovery

## 🔍 Overview

**Content discovery** is the process of finding resources on a web server that aren't directly linked or presented to the user — things not intended for public access but still reachable if you know where to look.

| Examples of Hidden Content | Why It Matters |
|---------------------------|---------------|
| Staff portals and admin panels | Direct access to privileged functionality |
| Backup files (`.bak`, `.old`, `.zip`) | May contain source code, credentials, or database dumps |
| Configuration files | Can expose API keys, database credentials, server settings |
| Old or deprecated pages | May run unpatched, vulnerable versions of the application |
| Development/staging endpoints | Often less secured than production |

> Content discovery sits at the heart of reconnaissance. The more of the attack surface you map, the more opportunities for vulnerabilities you find. Many critical findings — exposed admin panels, leaked credentials, unprotected APIs — come from content discovery rather than vulnerability scanning.

---

## 🔎 Manual Discovery

Manual discovery uses built-in web conventions and publicly accessible files to map the application without sending unusual traffic.

---

### 🤖 robots.txt

`robots.txt` is a file placed at the root of a website that instructs search engine crawlers which pages they are and aren't allowed to index. It is publicly accessible and not enforced — it is a *convention*, not a security control.

```
http://target.com/robots.txt
```

> **Why it's useful:** The `Disallow` entries are effectively a list of paths the site owner doesn't want publicised — admin panels, internal tools, backup directories. This is exactly what attackers want to find.

**Example `robots.txt`:**
```
User-agent: *
Disallow: /admin/
Disallow: /backup/
Disallow: /staff-portal/
Allow: /
```

---

### 🖼️ Favicon

The **favicon** is the small icon displayed in the browser tab and address bar. When a developer uses a framework and forgets to replace the default favicon, it reveals which framework the application was built with — and therefore which known vulnerabilities may apply.

**How to identify:**

```bash
# Step 1 — Find the favicon URL in the page source
# Open page source → Ctrl+F → search "favicon"

# Step 2 — Download and hash the favicon (Linux)
curl https://target.com/favicon.ico | md5sum

# Step 2 — Download and hash the favicon (PowerShell)
curl https://target.com/favicon.ico -UseBasicParsing -o favicon.ico
Get-FileHash .\favicon.ico -Algorithm MD5
```

**Step 3 — Look up the hash:**
→ [OWASP Favicon Database](https://wiki.owasp.org/index.php/OWASP_favicon_database)

Match the MD5 hash against the database to identify the framework.

---

### 🗺️ sitemap.xml

`sitemap.xml` is a file that lists every URL the site owner wants indexed by search engines. Unlike `robots.txt` which hides paths, the sitemap explicitly advertises them — but sometimes includes internal or legacy pages the owner forgot to remove.

```
http://target.com/sitemap.xml
```

> Sitemaps can reveal the full structure of a website — every page, post, and resource the owner has ever published. Old pages that have been removed from navigation but not from the sitemap may still be live.

---

### 📋 HTTP Headers

HTTP response headers often contain information about the server stack — web server software, scripting language, framework version, and more. This information can be used to identify vulnerable software versions.

```bash
# -v enables verbose mode — outputs full request and response headers
curl http://target.com -v
```

**Headers to look for:**

| Header | What It May Reveal |
|--------|-------------------|
| `Server` | Web server software and version — e.g. `nginx/1.18.0`, `Apache/2.4.49` |
| `X-Powered-By` | Backend language or framework — e.g. `PHP/7.4.3`, `ASP.NET` |
| `Set-Cookie` | Cookie names can hint at the framework — e.g. `PHPSESSID` = PHP, `JSESSIONID` = Java |
| `X-Generator` | CMS identifier — e.g. `WordPress 5.8` |

> Cross-reference any version numbers found against CVE databases (NVD, Exploit-DB) to identify known vulnerabilities.

---

## 🌍 OSINT — Open Source Intelligence

OSINT techniques discover publicly available information about a target without directly interacting with the target server.

---

### 🔎 Google Hacking / Dorking

**Google dorking** uses advanced Google search operators to surface specific content that is indexed but not easily found through normal searches.

| Operator | Description | Example |
|----------|-------------|---------|
| `site:` | Returns results only from the specified domain | `site:tryhackme.com` |
| `inurl:` | Results containing the specified word in the URL | `inurl:admin` |
| `intitle:` | Results with the specified word in the page title | `intitle:admin` |
| `filetype:` | Results with a specific file extension | `filetype:pdf` |
| `cache:` | Shows Google's cached version of a page | `cache:tryhackme.com` |
| `intext:` | Results containing specific text in the body | `intext:"index of /"` |

**Combined example:**
```
site:tryhackme.com filetype:pdf intitle:confidential
```

> **Google Hacking Database (GHDB):** Exploit-DB maintains a database of known effective dorks at `exploit-db.com/google-hacking-database` — categorised by type (login pages, sensitive files, vulnerable servers).

---

### 🔧 Wappalyzer

**Wappalyzer** identifies the technologies powering a website — frameworks, CMS, payment processors, analytics tools, CDNs, and more — by analysing HTTP headers, HTML, and JavaScript.

- **Website:** [wappalyzer.com](https://www.wappalyzer.com/)
- **Browser extension:** Available for Chrome and Firefox — runs passively as you browse

> Once you know the stack (e.g. WordPress 5.8, PHP 7.4, Apache 2.4), you can search for known CVEs and public exploits targeting those specific versions.

---

### ⏳ Wayback Machine

The **Wayback Machine** is an internet archive dating back to the 1990s, storing snapshots of websites over time.

- **URL:** [archive.org/web](https://archive.org/web/)

**How to use:** Enter a domain name to see all saved snapshots — browse any historical version of the site.

> **Why it's useful:** Pages that have been removed from a live site may still be accessible via old snapshots. Old endpoints, login pages, API documentation, and exposed files from previous versions of the site can reveal attack paths that still work on the current server.

---

### 🐙 GitHub

**GitHub** is a hosted version control platform. Developers often accidentally commit sensitive data — credentials, API keys, internal tools, and source code — to public repositories.

> **Git** tracks every change to a file over time — meaning even if a secret was added and then deleted in a later commit, it still exists in the repository's history and can be retrieved.

**What to search for:**
- Company name or domain — find public repos belonging to the target
- `config`, `env`, `.env`, `secret`, `password` — common filenames containing credentials
- Tool: **truffleHog**, **gitleaks** — scan repos for leaked secrets automatically

---

### 🪣 S3 Buckets

**Amazon S3** (Simple Storage Service) is a cloud storage service commonly used to host static files, backups, and assets. Misconfigured buckets are publicly readable — a common and critical misconfiguration.

**URL format:**
```
https://{bucket-name}.s3.amazonaws.com
```

**Examples:**
```
https://tryhackme-assets.s3.amazonaws.com
https://company-backup.s3.amazonaws.com
```

> **Finding buckets:** Try guessing bucket names based on the company name, product names, or domain. Tools like **S3Scanner** and **bucket-finder** automate this. A publicly readable bucket can expose database dumps, source code, internal documents, and more.

---

## ⚡ Automated Discovery

**Automated content discovery** uses tools to rapidly brute-force directories, files, and endpoints by sending large numbers of requests to the web server — testing each path in a wordlist to see what returns a valid response.

### Wordlists

Wordlists are text files containing large collections of common directory and file names. The most widely used collection is **SecLists** by Daniel Miessler.

```bash
# Common location on Kali / THM AttackBox
/usr/share/wordlists/SecLists/Discovery/Web-Content/common.txt
```

> **SecLists** on GitHub: `github.com/danielmiessler/SecLists`

---

### 🛠️ Tools

All three tools below perform the same core function — directory and file brute-forcing — with different syntax and features:

#### ffuf
```bash
ffuf -w /usr/share/wordlists/SecLists/Discovery/Web-Content/common.txt \
     -u http://target.com/FUZZ
```

| Flag | Description |
|------|-------------|
| `-w` | Path to the wordlist |
| `-u` | Target URL — `FUZZ` is replaced with each wordlist entry |
| `-fc` | Filter by HTTP status code — e.g. `-fc 404` hides 404s |
| `-mc` | Match by status code — e.g. `-mc 200,301` shows only hits |
| `-e` | File extensions to append — e.g. `-e .php,.txt,.html` |

---

#### dirb
```bash
dirb http://target.com/ /usr/share/wordlists/SecLists/Discovery/Web-Content/common.txt
```

Simple and beginner-friendly — runs automatically and outputs results in real time.

---

#### Gobuster
```bash
gobuster dir \
  --url http://target.com/ \
  -w /usr/share/wordlists/SecLists/Discovery/Web-Content/common.txt
```

| Flag | Description |
|------|-------------|
| `dir` | Directory/file brute-force mode |
| `--url` | Target URL |
| `-w` | Wordlist path |
| `-x` | File extensions — e.g. `-x php,txt,html` |
| `-t` | Number of concurrent threads — e.g. `-t 50` |
| `-o` | Output results to a file |

> **Choosing a tool:** ffuf is the most flexible and widely used in CTFs and professional assessments. Gobuster is fast and reliable for straightforward directory brute-forcing. dirb is the simplest to run for quick checks. All three produce similar results on the same wordlist.
