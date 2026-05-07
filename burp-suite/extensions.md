# Burp Suite — Extensions

## 🔍 Overview

**Extensions** allow you to customise and expand Burp Suite's functionality beyond its built-in modules. They can add new scanning capabilities, automate repetitive tasks, integrate with external tools, or introduce entirely new workflows — built by PortSwigger, the community, or yourself.

---

## 🖥️ Interface

### Extensions List

Displays all currently installed extensions — each can be individually activated or deactivated without uninstalling.

### Managing Extensions

| Action | Description |
|--------|-------------|
| **Add** | Install an extension from a local file (`.jar`, `.py`, `.rb`) — for extensions not available in the BApp Store |
| **Remove** | Uninstall the selected extension |
| **Up / Down** | Control the load order — determines the sequence in which extensions are invoked when processing requests and responses |

> **Load order matters:** Some extensions depend on or interact with others. If two extensions modify the same request, the one higher in the list processes it first.

### Details, Output & Errors

| Tab | Description |
|-----|-------------|
| **Details** | Metadata about the selected extension — name, description, version, author |
| **Output** | Live console output from the extension's execution — print statements, logs |
| **Errors** | Any errors or exceptions thrown during execution — essential for debugging custom extensions |

---

## 🏪 BApp Store

The **BApp Store** is PortSwigger's official marketplace for Burp Suite extensions — browse, install, and update extensions directly from within Burp.

| Detail | Description |
|--------|-------------|
| **Languages** | Most extensions are written in **Java** or **Python** — Python extensions require the Jython interpreter |
| **Access** | Extensions tab → BApp Store sub-tab |
| **Installation** | One-click install — Burp handles downloading and loading automatically |

> **Notable BApp extensions:** `Logger++` (enhanced request logging), `Autorize` (access control testing), `Active Scan++` (additional scan checks), `JWT Editor` (JWT manipulation), `Turbo Intruder` (high-speed request engine to replace rate-limited Community Intruder).

---

## 🐍 Jython — Python Extensions

**Jython** is a Java implementation of Python — it enables Burp Suite to run Python-based extensions inside the JVM.

### Setup

```
1. Download the Jython standalone JAR:
   → https://www.jython.org/download
   → Download: jython-standalone-x.x.x.jar

2. Configure Burp Suite:
   Extensions tab → Extension Settings sub-tab → Python Environment

3. Set the path:
   → "Location of Jython standalone JAR file"
   → Browse to and select the downloaded .jar file

4. Restart Burp Suite if prompted
```

> Once configured, Python-based BApp extensions install and run without any additional setup. JRuby can be configured the same way for Ruby extensions.

---

## 🔌 API — Building Custom Extensions

Burp Suite exposes an API that allows developers to build custom extensions — automating tasks, adding new tabs, modifying requests programmatically, or integrating with external tools.

| Language | Requirement |
|----------|-------------|
| **Java** | Native — no additional setup needed |
| **Python** | Requires Jython (configured above) |
| **Ruby** | Requires JRuby |

**Getting started:**

→ [Writing Your First Burp Suite Extension](https://portswigger.net/burp/extender/writing-your-first-burp-suite-extension)

> **What the API enables:** Custom extensions can hook into Burp's request/response pipeline, add custom scan checks, create new UI tabs, log specific patterns, automate payload generation, or interact with any Burp module programmatically. This is how tools like **Turbo Intruder** and **Autorize** are built.
