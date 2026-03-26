#  Full-Lifecycle Cyber Attack Simulation & SOC Telemetry Mapping

## Executive Summary
This project simulates a complete, end-to-end cyber attack lifecycle within an isolated lab environment. By executing a coordinated attack from a Kali Linux machine against a Metasploitable2 victim server, this exercise demonstrates the progression from initial reconnaissance to full system compromise. 

Crucially, this project adopts a **Purple Team methodology**. For every offensive action taken, the project maps the behavior to the **MITRE ATT&CK Framework** and highlights the corresponding defensive telemetry (logs, traffic patterns, and alerts) that a Security Operations Center (SOC) would use to detect and contain the threat.

##  Tech Stack & Frameworks
* **Offensive Tools:** Kali Linux, Metasploit Framework, Nmap, Burp Suite (implied/utilized for web exploitation)
* **Target Infrastructure:** Metasploitable2 (Linux), Damn Vulnerable Web App (DVWA), vsFTPd, Samba
* **Security Frameworks:** MITRE ATT&CK (T1046, T1190, T1210)
* **Defensive Concepts:** Web Server Logging, System Auditing, Network Traffic Analysis

---

##  System Architecture & Threat Model

The environment consists of an attacker machine and a vulnerable server operating on a shared, isolated subnet. 

<p align="center">
  <img src=".assets/Architecture Diagram.png" alt="Architecture Diagram and Attack Flow" width="850"/>
  <br>
  <b>Figure 1: Attack Architecture and Detection Mechanisms</b>
</p>

### The Attack Flow & Telemetry Generation:
The diagram above illustrates the relationship between the attacker, the exposed attack surface, and the detection mechanisms. 
* **Attack Surface:** Exposed web applications (DVWA) and vulnerable internal services (FTP, SMB).
* **Detection Pipeline:** Every phase of the attack generates observable telemetry—SQL injection attempts are captured in web server logs, privilege escalation is recorded in system logs, and C2 beacons are detected via network monitoring.

---

##  Phase 1: Reconnaissance & Surface Discovery

The attack chain initiates with comprehensive network and service enumeration to identify potential entry points.

### Service Enumeration
A full port scan was executed against the target, revealing multiple exposed services including HTTP (Port 80), FTP (Port 21 running vsFTPd 2.3.4), SSH (Port 22), and Samba (Ports 139/445). 

<p align="center">
  <img src=".assets/Nmap full port scan.png" alt="Nmap Full Port Scan" width="750"/>
  <br>
  <b>Figure 2: Nmap Service Enumeration Results</b>
</p>

* **MITRE ATT&CK Mapping:** `T1046 - Network Service Discovery`
* **SOC Telemetry:** High-volume anomalous inbound connection requests from a single IP across multiple ports (Network Flow Logs).

---

##  Phase 2: Application Layer Exploitation (DVWA)

Following reconnaissance, the attacker targets the web application layer to bypass authentication and exfiltrate data.

### 1. SQL Injection (SQLi) & Data Extraction
Authentication bypass was achieved using malicious input (`' OR '1'='1`), allowing unauthorized access. Further exploitation utilized UNION-based queries to extract the backend database schema, revealing critical tables such as `users` and `user_permissions`.

<p align="center">
  <img src=".assets/SQL Injection - Table Enumeration.png" alt="SQL Injection Table Enumeration" width="800"/>
  <br>
  <b>Figure 3: UNION-based SQLi Database Enumeration</b>
</p>
<p align="center">
  <img src=".assets/SQL Injection - Data Extraction.png" alt="SQL Injection Data Extraction" width="800"/>
  <br>
  <b>Figure 4: Data Exfiltration via SQL Injection</b>
</p>

* **MITRE ATT&CK Mapping:** `T1190 - Exploit Public-Facing Application`
* **SOC Telemetry:** Anomalous SQL syntax in HTTP GET/POST parameters (WAF/Web Access Logs).

### 2. Cross-Site Scripting (XSS) & Session Hijacking
In parallel, a reflected XSS payload was injected into the application. This successfully executed malicious JavaScript in the victim's browser, exfiltrating the `PHPSESSID` cookie. The attacker then reused this cookie to impersonate the authenticated user, achieving full account takeover.

<p align="center">
  <img src=".assets/XSS - Cookie Theft.png" alt="XSS Cookie Theft" width="800"/>
  <br>
  <b>Figure 5: XSS Payload Executing Cookie Theft</b>
</p>
<p align="center">
  <img src=".assets/XSS - Session Hijack .png" alt="Session Hijacking" width="800"/>
  <br>
  <b>Figure 6: Successful Session Impersonation</b>
</p>

* **MITRE ATT&CK Mapping:** `T1539 - Steal Session Cookie` / `T1550 - Use Alternate Authentication Material`

---

##  Phase 3: System-Level Compromise & RCE

With web layer access established, the attacker escalates to remote code execution (RCE) at the OS level using the Metasploit Framework.

### vsFTPd & Samba Exploitation
The attacker targeted the `vsFTPd 2.3.4` backdoor vulnerability, successfully popping a command shell. Privilege escalation to `root` was instantly confirmed via the `whoami` command. Alternatively, the `Samba username map script` vulnerability was exploited to achieve a secondary path to remote command execution.

<p align="center">
  <img src=".assets/Metasploitable - Root access proof.png" alt="Root Access via vsFTPd" width="750"/>
  <br>
  <b>Figure 7: Root Shell Achieved via vsFTPd Backdoor</b>
</p>
<p align="center">
  <img src=".assets/Metasploitable - Samba exploit .png" alt="Samba Exploitation" width="750"/>
  <br>
  <b>Figure 8: Alternative RCE via Samba Vulnerability</b>
</p>

* **MITRE ATT&CK Mapping:** `T1210 - Exploitation of Remote Services`
* **SOC Telemetry:** Shell processes spawning from service accounts (e.g., `/bin/sh` executed by `ftp` or `smbd` daemons) logged in EDR/Syslog.

---

##  Phase 4: Post-Exploitation & Lateral Verification

To validate full operational control, the compromised machine was instructed to initiate outbound network requests.

### Command Execution & Connectivity Validation
The attacker executed outbound `ping` commands from the victim server targeting both the attacker's Kali IP and external infrastructure (Google). This confirms active Command and Control (C2) capability and verifies that the victim machine can route traffic outbound for potential data exfiltration.

<p align="center">
  <img src=".assets/Metasploitable VM - Kali IP ping.png" alt="Ping to Attacker Machine" width="750"/>
</p>
<p align="center">
  <img src=".assets/Metasploitable VM - Google ping.png" alt="Ping to External Network" width="750"/>
  <br>
  <b>Figure 9 & 10: Validation of Outbound Connectivity and Command Execution</b>
</p>

* **MITRE ATT&CK Mapping:** `T1059 - Command and Scripting Interpreter`
* **SOC Telemetry:** Unexpected outbound ICMP/TCP traffic originating from a hardened server to unknown external IPs (Firewall/VPC Flow Logs).

---

##  Strategic Impact & Security Outcomes

* **Comprehensive Attack Surface Validation:** Demonstrated how attackers chain multiple low-level vulnerabilities (XSS, SQLi, outdated daemons) to achieve total infrastructure compromise.
* **Telemetry Correlation:** Highlighted the importance of centralizing logs. Detecting this attack requires correlating Web Logs (Phase 2), Syslog/Auditd (Phase 3), and Network Flow Logs (Phase 1 & 4).
* **Framework Mastery:** Successfully mapped the entire cyber kill chain to the industry-standard MITRE ATT&CK framework, proving a mature understanding of adversary tradecraft and defensive posturing.
