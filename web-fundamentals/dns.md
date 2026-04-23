# DNS — Domain Name System

## 🌐 Overview

**DNS** translates human-readable domain names into IP addresses — so you can type `tryhackme.com` instead of memorising `104.26.10.229`. It acts as the internet's phone book, mapping names to numbers at massive scale.

> **Why it matters for security:** DNS is fundamental to almost every network interaction. Attackers abuse it for C2 communication, data exfiltration (DNS tunnelling), phishing (typosquatting domains), and reconnaissance. Understanding how it works is the foundation for understanding how it breaks.

---

## 🏗️ Domain Hierarchy

Domains are structured in a hierarchy from right to left — the rightmost label is the most general, the leftmost is the most specific.

```
admin.tryhackme.com
  │        │       └── TLD (Top-Level Domain)
  │        └────────── Second-Level Domain
  └─────────────────── Subdomain
```

| Level | Description | Example |
|-------|-------------|---------|
| **TLD** (Top-Level Domain) | Rightmost part of the domain — managed by ICANN | `.com`, `.org`, `.io`, `.co.uk` |
| **Second-Level Domain** | The registered, human-chosen name — unique within its TLD | `tryhackme` in `tryhackme.com` |
| **Subdomain** | Left of the second-level domain — can be nested multiple levels deep | `admin` in `admin.tryhackme.com` |

> **TLD types:** Generic TLDs (gTLDs) like `.com`, `.net`, `.org` are globally managed. Country-code TLDs (ccTLDs) like `.co.uk`, `.de`, `.fr` are administered by individual countries. New gTLDs like `.hacker`, `.app`, and `.cloud` have been introduced in recent years.

---

## 📋 Record Types

Each DNS record type serves a specific purpose. A single domain can have multiple records of different types simultaneously.

| Record | Purpose | Example |
|--------|---------|---------|
| **A** | Maps a domain to an **IPv4** address | `tryhackme.com` → `104.26.10.229` |
| **AAAA** | Maps a domain to an **IPv6** address | `tryhackme.com` → `2606:4700:20::681a:be5` |
| **CNAME** | Maps a domain to **another domain name** — not an IP | `store.tryhackme.com` → `shops.shopify.com` |
| **MX** | Points to the **mail server(s)** that handle email for the domain | `tryhackme.com` → `alt1.aspmx.l.google.com` |
| **TXT** | Stores **arbitrary text data** — used for domain verification, SPF, DKIM, DMARC | `v=spf1 include:mailgun.org ~all` |
| **NS** | Specifies the **authoritative nameservers** for the domain | `tryhackme.com` → `kip.ns.cloudflare.com` |
| **PTR** | **Reverse DNS** — maps an IP address back to a domain name | `104.26.10.229` → `tryhackme.com` |

### Record Notes

- **CNAME chains:** A CNAME points to another name, not an IP — the resolver follows the chain until it reaches an A or AAAA record. You cannot have a CNAME at the root of a domain (the "apex"), only on subdomains.
- **MX priority:** MX records have a numerical priority value — lower numbers are tried first. Multiple MX records provide redundancy (primary + backup mail servers).
- **TXT security uses:** SPF records tell mail servers which IPs are authorised to send email for the domain. DKIM adds a cryptographic signature to verify email authenticity. DMARC tells receivers what to do when SPF or DKIM fails. These are all stored as TXT records.

---

## 🔄 DNS Request Process

When you visit a website, a DNS resolution chain runs behind the scenes — typically completing in milliseconds.

```
Your Computer → Recursive DNS Server → Root DNS Servers
                                             ↓
                                       TLD DNS Server
                                             ↓
                                   Authoritative DNS Server
                                             ↓
                              Answer returned and cached at each step
```

**Step-by-step:**

**1 — Local Cache Check**
Your computer first checks its own local DNS cache. If the domain was recently resolved, the cached answer is used immediately and no network request is made. Operating systems cache DNS responses for the duration of the record's TTL (Time to Live).

**2 — Recursive DNS Server**
If not cached locally, the request goes to your **Recursive DNS Server** — typically provided by your ISP or a public resolver (e.g. `8.8.8.8` Google, `1.1.1.1` Cloudflare). This server also checks its own cache before proceeding. If found, it returns the cached answer.

**3 — Root DNS Servers**
If not cached, the recursive server queries one of the **13 root DNS server clusters** — the backbone of the DNS system. The root server doesn't know the final answer, but it knows which TLD server to ask. It returns a referral: *"Go ask the `.com` TLD servers."*

**4 — TLD DNS Server**
The **TLD server** holds records for all domains registered under that TLD. It doesn't know the IP either, but it knows which **authoritative nameserver** is responsible for the domain. It returns the nameserver details — e.g. `kip.ns.cloudflare.com` and `uma.ns.cloudflare.com` for `tryhackme.com` (the second serves as a backup).

**5 — Authoritative DNS Server**
The **authoritative nameserver** is the definitive source for the domain's DNS records. This is where all A, AAAA, MX, TXT, and other records are stored, and where updates must be made. It returns the final answer — the resolved record — to the recursive server.

**6 — Response and Caching**
The recursive DNS server caches the answer for the duration of the record's TTL, then relays it back to your computer, which also caches it locally. Future requests for the same domain skip the entire chain until the TTL expires.

> **TTL (Time to Live):** Every DNS record has a TTL value in seconds — e.g. `300` means the record can be cached for 5 minutes. Low TTLs allow faster propagation of changes; high TTLs reduce DNS query load but slow down updates. Attackers use very low TTLs for DNS-based C2 (fast-flux DNS) to rotate IPs rapidly.

---

## 💻 Commands

### nslookup

`nslookup` queries DNS records from the command line. Use `--type=` to specify the record type.

| Command | Description |
|---------|-------------|
| `nslookup --type=A tryhackme.com` | Look up the IPv4 address for a domain |
| `nslookup --type=AAAA tryhackme.com` | Look up the IPv6 address for a domain |
| `nslookup --type=CNAME shop.website.thm` | Find the CNAME target for a subdomain |
| `nslookup --type=MX website.thm` | Find the mail server(s) and their priority values for a domain |
| `nslookup --type=TXT website.thm` | Retrieve TXT records — useful for SPF, DKIM, domain verification tokens |
| `nslookup --type=NS website.thm` | Find the authoritative nameservers for a domain |

> **MX priority:** The output includes a numerical priority value — lower numbers are preferred. If the primary mail server is unavailable, the next lowest priority is tried.

### dig (alternative)

`dig` provides more detailed output and is preferred in many security contexts:

```bash
dig tryhackme.com A          # IPv4 record
dig tryhackme.com MX         # Mail servers
dig tryhackme.com TXT        # TXT records
dig tryhackme.com NS         # Nameservers
dig +short tryhackme.com A   # Clean output — IP only
dig @8.8.8.8 tryhackme.com   # Query a specific DNS resolver (Google)
```
