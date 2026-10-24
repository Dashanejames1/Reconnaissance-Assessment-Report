# 📋 Recon-Assessment-Report

**Author:** Dashane James  
**Lab Environment:** VMware Workstation | Kali Linux | Metasploitable 2  
**Purpose:** The goal of this repository is to translate raw tool output into analyst-grade documentation. Specifically, the ability to prioritize vulnerabilities by severity, justify each ranking, and communicate impact and remediation in language that drives a business decision rather than just describing a technical finding. 
**Status:** 🔵 Completed

---

## 📋 Overview

This repository documents the final Week 1 assessment report for my CySA+ CS0-003 lab portfolio. Over 6 days I conducted a full passive-to-active reconnaissance cycle against Metasploitable 2 (192.168.79.130) using Nmap, Wireshark, TCPDump, and the Nmap Scripting Engine. This report compiles all findings from those individual labs into a single ranked assessment — 8 confirmed vulnerabilities ranging from CVSS 5.0 (Medium) to CVSS 9.8 (Critical), including one finding that was actively exploited live during the assessment.

---

## 🧪 Lab Environment

| Component | Details |
|---|---|
| Hypervisor | VMware Workstation (Host-Only Network) |
| Attacker Machine | Kali Linux 2026.1 — `192.168.79.129` |
| Target Machine | Metasploitable 2 — `192.168.79.130` |
| Network Type | Host-Only (isolated, no internet exposure) |
| Host OS | Windows 11 — ASUS Vivobook 14 |
| Assessment Scope | Days 1–6 recon and vulnerability detection labs |

> ⚠️ **Note:** All activity was performed in a controlled, isolated lab environment against deliberately vulnerable machines. No unauthorized access to live networks was performed.

---

## 🛠️ Tools Used

- **Nmap** — Host discovery, port scanning (SYN/UDP/full TCP), service version detection, OS fingerprinting, and NSE vulnerability scripting
- **Wireshark** — GUI-based packet capture on eth0; traffic filtering and Follow TCP Stream analysis
- **TCPDump** — CLI-based packet capture; host filtering, .pcap file saves, and plaintext Telnet session capture via -A flag
- **NVD (nvd.nist.gov)** — CVE research and CVSS score verification for identified software versions

---

## 🔬 Tasks / Assessments Performed

### 1. Compile host discovery and port scan results from Days 1–2.

I reviewed my baseline scan results from Day 1 and the scan type comparison from Day 2. The goal was to confirm the full attack surface of Metasploitable 2 before moving into service-level analysis.

```bash
# Day 1 — Ping sweep (3 live hosts found)
sudo nmap -n --disable-arp-ping -sP 192.168.79.0/24

# Day 1 — Fast scan to baseline 100 top ports
sudo nmap -n --disable-arp-ping -F 192.168.79.130 -oN ~/baseline.txt

# Day 2 — SYN half-open scan (confirmed 18 open ports)
sudo nmap -sS -Pn --disable-arp-ping -n -F 192.168.79.130

# Day 2 — Full TCP connect scan across all 65,535 ports
sudo nmap -sT -Pn --disable-arp-ping -n -p- 192.168.79.130 -oN ~/full_scan.txt
```

**Findings:**
The target had 18 open TCP ports: 21, 22, 23, 25, 53, 80, 111, 139, 445, 513, 514, 2049, 2121, 3306, 5432, 5900, 6000, and 8009. The full port scan completed in 5.36 seconds versus 0.26 seconds for the fast scan, confirming why full scans are reserved for thorough assessments. Three live hosts were identified on the network — the gateway (.1), the attacker (.129), and the target (.130).

---

### 2. Map service versions to CVEs using Nmap -sV and NVD research.

I ran version detection against the highest-risk ports identified in Days 1–2 to get exact software versions. I then cross-referenced each version against the National Vulnerability Database (NVD) to identify known CVEs.

```bash
# Version detection against key ports
sudo nmap -sV -Pn --disable-arp-ping -n -p 21,22,23,80,3306 192.168.79.130

# OS fingerprinting
sudo nmap -O -Pn --disable-arp-ping -n 192.168.79.130
```

Output — confirmed service versions:

| Port | Service | Version |
|---|---|---|
| 21/tcp | FTP | vsftpd 2.3.4 |
| 22/tcp | SSH | OpenSSH 4.7p1 Debian 8ubuntu1 (protocol 2.0) |
| 23/tcp | Telnet | Linux telnetd |
| 80/tcp | HTTP | Apache httpd 2.2.8 ((Ubuntu) DAV/2) |
| 3306/tcp | MySQL | MySQL 5.0.51a-3ubuntu5 |

OS fingerprinting returned no exact match — raw TCP/IP fingerprint only. The Debian service strings in the banners confirm a Linux-based target.

**NVD Research — vsftpd 2.3.4:**
Searching nvd.nist.gov for "vsftpd 2.3.4" returned CVE-2011-2523 with a CVSS v3.1 score of 9.8 (Critical). Vector: AV:N/AC:L/PR:N/UI:N/S:U/C:H/I:H/A:H. This CVE describes an intentional backdoor that allows a remote attacker to receive a root shell with no credentials. This became the highest-priority finding in the entire assessment.

**Finding:**
Version detection is essential because it converts an open port into a targetable vulnerability. Without -sV, port 21 is just "ftp open" — with it, it becomes vsftpd 2.3.4, which maps to a 9.8 Critical CVE. Every major finding in this report originated from a service version confirmed by -sV.

---

### 3. Confirm cleartext credential exposure via Wireshark and TCPDump.

I used Wireshark's Follow TCP Stream feature on a port 80 HTTP session to confirm that credentials and server information were transmitted in plaintext. I then confirmed Telnet's plaintext behavior independently using TCPDump's -A flag, without needing a GUI.

```bash
# Day 4 — Wireshark filter to isolate target traffic
# Filter: ip.addr == 192.168.79.130
# Then: Right-click TCP packet on port 80 → Follow → TCP Stream

# Day 5 — TCPDump Telnet plaintext capture
sudo tcpdump -i eth0 port 23 -A -n
```

**Day 4 Findings — Wireshark:**
Following the TCP stream on port 80 revealed the complete HTTP conversation in plaintext. The response body contained the Apache/PHP version banners, developer contact information, and default login credentials (msfadmin/msfadmin) — all readable without any decryption. The page also listed Metasploitable's hosted web applications: DVWA, Mutillidae, phpMyAdmin, TWiki, and WebDAV.

**Day 5 Findings — TCPDump:**
Running TCPDump with the -A flag while a Telnet session was active captured the live session contents directly in the terminal. The shell prompt msf@metasploitable was visible in the ASCII output, confirming that Telnet transmits session data completely unencrypted — no GUI or stream reconstruction needed. This proves why Telnet should never be used in any environment and must be replaced with SSH.

---

### 4. Run NSE vulnerability scripts and document all results.

On Day 6 I ran a progression of Nmap scripts — default scripts (-sC), targeted service checks (ftp-anon, smb-vuln*), and then a broad vulnerability scan across all open ports (--script vuln). The broad vuln scan produced the most significant finding of the entire week.

```bash
# Default scripts — ports 21, 22, 80, 445
sudo nmap -sC -Pn -n -p 21,22,80,445 192.168.79.130

# Targeted FTP anonymous login check
sudo nmap --script ftp-anon -Pn --disable-arp-ping -n -p 21 192.168.79.130

# SMB vulnerability scripts
sudo nmap --script smb-vuln* -Pn --disable-arp-ping -n -p 445 192.168.79.130

# Broad vulnerability scan — all open ports
sudo nmap --script vuln 192.168.79.130
```

**Default script results:**
Port 21 — Anonymous FTP login allowed (FTP response code 230). Port 445 — Samba 3.0.20-Debian; message_signing: disabled (dangerous, but default); guest authentication: allowed. The SMB-specific vuln scripts confirmed MS10-061 and MS10-054 as not vulnerable; smb-vuln-regsvc-dos returned inconclusive (script error, not a definitive result).

**Broad vuln scan — critical result:**
NSE --script vuln on port 21 triggered the vsftpd 2.3.4 backdoor (CVE-2011-2523) and returned:

```
uid=0(root) gid=0(root) groups=0(root)
```

This is the only finding in Week 1 that went from detection all the way to confirmed live exploitation. A root shell was returned by Nmap's script with no credentials and no manual exploit steps. All other findings were detected — this one was proven.

**Finding:**
There is a meaningful difference between a vulnerability being detected and a vulnerability being exploited. CVE-2011-2523 was both — flagged by the NSE script and exploited in the same scan. This is the scenario that would trigger an immediate Incident Response engagement in a real environment.

---

### 5. Rank all findings by CVSS score and document impact and remediation.

I compiled all 8 confirmed findings, assigned CVSS scores (sourced from NVD where available, estimated from the vulnerability class where not), and ranked them highest to lowest. For tied scores, I ranked by exploitability — how much attacker knowledge is required to trigger the vulnerability.

```bash
# Ranking tiebreaker logic:
# - Equal CVSS → more direct attack path ranked higher
# - FTP Anonymous (7.5) ranks above Apache 2.2.8 (7.5) because
#   anonymous access requires zero attacker knowledge (no CVE lookup),
#   whereas targeting Apache CVE-2011-3368 requires knowing the specific version
```

**Findings:**
Ranking by CVSS revealed that 5 of the 8 findings are exploitable with no authentication and low attack complexity (AV:N/AC:L/PR:N). The three medium findings still carry meaningful risk — SMB signing disabled enables MITM attacks, OpenSSH 4.7p1 supports weak cipher suites, and MySQL 3306 being network-accessible is a configuration failure regardless of the version. No finding ranked below Medium.

---


## 📊 Key Findings Summary

| Rank | Port/Service | CVE | CVSS | Risk Level | Confirmed Via |
|---|---|---|---|---|---|
| 1 | 21/tcp — vsftpd 2.3.4 | CVE-2011-2523 | 9.8 | 🔴 Critical | Nmap -sV, NVD, NSE vuln (EXPLOITED — uid=0) |
| 2 | 80/tcp — SQL Injection | CWE-89 | ~8.8 | 🔴 Critical | NSE --script vuln, Wireshark TCP Stream |
| 3 | 21/tcp — FTP Anonymous Login | CWE-521 | ~7.5 | 🟠 High | NSE ftp-anon (FTP code 230) |
| 4 | 80/tcp — Apache httpd 2.2.8 | CVE-2011-3368 | ~7.5 | 🟠 High | Nmap -sV, Wireshark (banner + creds in HTTP) |
| 5 | 23/tcp — Telnet Cleartext | N/A (protocol) | ~7.4 | 🟠 High | Wireshark Follow Stream, TCPDump -A (msf@metasploitable) |
| 6 | 445/tcp — SMB Signing Disabled | CVE-2011-2694 | ~6.8 | 🟡 Medium | NSE -sC (message_signing: disabled, guest auth) |
| 7 | 22/tcp — OpenSSH 4.7p1 Weak Crypto | CVE-2008-5161 | 5.9 | 🟡 Medium | Nmap -sV, NSE --script vuln |
| 8 | 3306/tcp — MySQL 5.0.51a Info Disclosure | CVE-2008-0226 | ~5.0 | 🟡 Medium | Nmap -sV, NSE --script vuln |

**Risk Levels:** 🔴 Critical | 🟠 High | 🟡 Medium | 🟢 Low

---

## 🗺️ MITRE ATT&CK Mapping

| Action Performed | ATT&CK Tactic | Technique ID | Technique Name |
|---|---|---|---|
| Nmap ping sweep to identify live hosts | Reconnaissance | T1595.001 | Scanning IP Blocks |
| Nmap SYN/UDP/full TCP port scans | Reconnaissance | T1595.002 | Vulnerability Scanning |
| Nmap -sV service banner grabbing | Reconnaissance | T1046 | Network Service Discovery |
| Nmap -O OS fingerprinting | Reconnaissance | T1592 | Gather Victim Host Information |
| Wireshark / TCPDump traffic capture | Credential Access | T1040 | Network Sniffing |
| Follow TCP Stream — msfadmin/msfadmin in HTTP | Credential Access | T1040 | Network Sniffing |
| NSE --script vuln broad vulnerability scan | Reconnaissance | T1595.002 | Vulnerability Scanning |
| Scan output saved to file (-oN, -w .pcap) | Collection | T1005 | Data from Local System |
| vsftpd backdoor triggered via NSE — uid=0(root) | Execution | T1059 | Command Execution |
| Root shell achieved without prior access | Privilege Escalation | T1068 | Exploitation for Privilege Escalation |

---

## 🛡️ Defensive Recommendations

Based on findings, the following remediations would be recommended in a real environment:

1. **vsftpd 2.3.4 (Critical — Exploited)** — Immediately uninstall vsftpd 2.3.4. This version contains an intentional backdoor and must never run in any environment. Replace with a patched FTP daemon or migrate to SFTP over SSH.

2. **SQL Injection on Port 80 (High)** — Implement parameterized queries and input validation across all web application forms. Deploy a Web Application Firewall (WAF) and update all web frameworks running on the server.

3. **FTP Anonymous Login (High)** — Disable anonymous FTP access in the vsftpd configuration (anonymous_enable=NO). Require authenticated accounts for all FTP sessions.

4. **Apache httpd 2.2.8 — EOL (High)** — Upgrade to a supported Apache release (2.4.x or later). Suppress server banners (ServerTokens Prod; ServerSignature Off). Enforce HTTPS/TLS — eliminate all unencrypted HTTP.

5. **Telnet Enabled (High)** — Disable Telnet immediately and replace with SSH for all remote access. SSH encrypts session data end-to-end; Telnet transmits everything in plaintext, including credentials.

6. **SMB Signing Disabled — Samba 3.0.20 (Medium)** — Enable SMB message signing (server signing = mandatory in smb.conf). Disable guest access. Restrict port 445 to authorized hosts via host-based firewall rules.

7. **OpenSSH 4.7p1 — Weak Crypto (Medium)** — Upgrade OpenSSH to a current release. In sshd_config, disable weak algorithms (RC4, DES, MD5) and enforce strong ciphers (AES-256-GCM, ChaCha20-Poly1305) only.

8. **MySQL 5.0.51a — Network-Exposed (Medium)** — Bind MySQL to localhost only (bind-address=127.0.0.1 in my.cnf). Upgrade to a supported MySQL or MariaDB version. A database port should never be directly accessible on a network interface.

---

## 🔑 Technical Notes

```bash
# ── DAY 1 — Host Discovery & Baseline ──────────────────────────────────
# Ping sweep — find live hosts
sudo nmap -n --disable-arp-ping -sP 192.168.79.0/24

# Fast scan — top 100 ports, save to file
sudo nmap -n --disable-arp-ping -F 192.168.79.130 -oN ~/baseline.txt

# ── DAY 2 — Scan Types ─────────────────────────────────────────────────
# SYN half-open scan (requires root; faster and stealthier than -sT)
sudo nmap -sS -Pn --disable-arp-ping -n -F 192.168.79.130

# UDP top-20 scan (slowest; only way to find UDP-only services)
sudo nmap -sU -Pn --disable-arp-ping -n --top-ports 20 192.168.79.130

# Full TCP connect scan — all 65,535 ports
sudo nmap -sT -Pn --disable-arp-ping -n -p- 192.168.79.130 -oN ~/full_scan.txt

# ── DAY 3 — Version Detection & CVE Research ───────────────────────────
# Service version detection on key ports
sudo nmap -sV -Pn --disable-arp-ping -n -p 21,22,23,80,3306 192.168.79.130

# OS fingerprinting
sudo nmap -O -Pn --disable-arp-ping -n 192.168.79.130

# CVE research: search nvd.nist.gov for each identified software version

# ── DAY 4 — Wireshark ──────────────────────────────────────────────────
# Wireshark display filter — isolate target traffic
ip.addr == 192.168.79.130

# To read a full application-layer session (HTTP, Telnet, etc.):
# Right-click a TCP packet → Follow → TCP Stream

# ── DAY 5 — TCPDump ────────────────────────────────────────────────────
# Filtered live capture — host only
sudo tcpdump -i eth0 host 192.168.79.130

# Save to .pcap for Wireshark analysis
sudo tcpdump -i eth0 -w capture.pcap
wireshark ~/capture.pcap

# Capture Telnet in readable ASCII (no decryption needed)
sudo tcpdump -i eth0 port 23 -A -n

# ── DAY 6 — NSE ────────────────────────────────────────────────────────
# Default script scan
sudo nmap -sC -Pn -n -p 21,22,80,445 192.168.79.130

# Targeted FTP anonymous login check
sudo nmap --script ftp-anon -Pn --disable-arp-ping -n -p 21 192.168.79.130

# SMB vulnerability scripts
sudo nmap --script smb-vuln* -Pn --disable-arp-ping -n -p 445 192.168.79.130

# Broad vulnerability scan — all open ports
sudo nmap --script vuln 192.168.79.130

# ── IMPORTANT NOTES ────────────────────────────────────────────────────
# Always add -n to Nmap scans in VMware environments to prevent DNS
# resolution hangs on the isolated host-only network.

# -sS requires root (raw packet crafting). -sT does not, but full
# connections are more likely to be logged by the target.

# --top-ports 20 is one unbroken flag — "--top- ports 20" is invalid syntax.

# TCPDump -A flag displays packet payload in ASCII — Telnet sessions
# are readable directly in the terminal without any decryption or GUI.

# Wireshark filter syntax: always use ip.addr == (double equals).
# A bare IP address without a field comparison is invalid syntax.
```

---


**The Overall Lesson**

The lesson of Week 1 is that reconnaissance is not one tool — it is a workflow. No single lab in Week 1 would have produced the full picture on its own. Nmap shows you what is there. Version detection shows you what it is. CVE research shows you what risk that creates. Packet capture shows you what is actually happening at the protocol level. NSE scripting shows you whether the theoretical vulnerability is real. And the report ties it all together into something you can hand to another person.

Every tool in a security analyst's kit exists to answer a different question about the same environment. The job is not to run a tool — it is to know which question to ask next, and which tool answers it. Week 1 built that instinct deliberately, one layer at a time, starting with "what is alive on this network" and ending with a root shell and a written report of how it happened. That is the workflow an analyst is expected to understand. Not just what each tool does but what role each tool plays in a larger investigation.

## 📌 About This Project

This repository is the capstone report for my Week 1 CySA+ lab portfolio. It synthesizes findings from six individual reconnaissance labs into a structured analyst report, demonstrating the full recon-to-documentation workflow: active scanning → version detection → CVE research → traffic analysis → exploitation confirmation → CVSS ranking → remediation. The format mirrors real-world security assessment deliverables including Security Operations, Vulnerability Management, and Reporting.

Related repositories:

- `Nmap-Host-Discovery-and-Lab-Baseline` — Established the lab network baseline and identified 3 live hosts and 18 open ports using ping sweep and fast scan
- `Nmap-Scan-Types-SYN-vs-TCP-vs-UDP` — Compared SYN, TCP connect, and UDP scan types, documenting speed, stealth, and completeness tradeoffs
- `Service-Version-Detection-and-OS-Fingerprinting` — Identified exact software versions on Metasploitable and cross-referenced against NVD for CVE-2011-2523
- `Wireshark-Capture-and-Analyze-Traffic` — Captured HTTP traffic and confirmed msfadmin/msfadmin credentials in cleartext via Follow TCP Stream
- `TCPDump-CLI-Packet-Capture` — Captured a live Telnet session in ASCII using the -A flag, proving cleartext exposure without a GUI
- `Nmap-Scripting-Engine` — Ran default, targeted, and broad NSE scans; triggered CVE-2011-2523 and confirmed uid=0(root) root shell via exploitation

---

## 👤 Author

**Dashane James**  
Senior Field Service Technician → Cybersecurity Analyst  
📍 Yonkers, NY  
🎓 B.S. Information Technology — SUNY Canton  
🏆 CompTIA Security+ | CySA+ (In Progress)  
🔗 [GitHub](https://github.com/Dashanejames1) | [Zero Trust Cyber Security Brand](https://www.instagram.com/zerotrust_cybersecurity)

---

*This repository is part of an active portfolio demonstrating hands-on cybersecurity skills. All lab work performed in isolated environments for educational purposes.*
