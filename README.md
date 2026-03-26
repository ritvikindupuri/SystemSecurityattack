# Full-Lifecycle Cyber Attack Simulation & MITRE ATT&CK Mapping

## Executive Summary
This project simulates a complete, end-to-end cyber attack lifecycle within an isolated virtualization lab. By executing a coordinated attack from a Kali Linux machine against a Metasploitable2 victim server, this exercise demonstrates the progression from initial reconnaissance to full system compromise. 

The project emphasizes realistic offensive attack chaining—moving from external service discovery, to application-layer exploitation, and ultimately escalating to OS-level root compromise—while mapping every tactical action to the industry-standard **MITRE ATT&CK Framework**. 

## Tech Stack & Frameworks
* **Offensive Tools:** Kali Linux, Metasploit Framework, Nmap, Burp Suite 
* **Target Infrastructure:** Metasploitable2 (Linux), Damn Vulnerable Web App (DVWA), vsFTPd, Samba
* **Virtualization:** Nutanix AHV Cluster
* **Security Frameworks:** MITRE ATT&CK (T1046, T1190, T1539, T1210)

---

## System Architecture & Threat Model

The environment consists of an attacker machine and a vulnerable server operating on a shared, isolated Nutanix AHV subnet. 

<p align="center">
  <img src=".assets/Architecture Diagram.png" alt="Architecture Diagram and Attack Flow" width="850"/>
  <br>
  <b>Figure 1: Attack Architecture and Exploitation Flow</b>
</p>

### The Attack Flow:
The diagram above illustrates the relationship between the attacker and the exposed attack surface. The target machine (`44.67.49.35`) hosts several intentionally vulnerable services, allowing the simulation to progress systematically from external discovery to application abuse and internal remote code execution.

---

## Phase 1: Environment Setup & Connectivity Verification

Before initiating the offensive attack chain, baseline network connectivity was established and verified to ensure the isolated lab environment was routing traffic correctly. 

### Network Validation
The Metasploitable2 victim machine was instructed to execute outbound `ping` commands. This verified two critical routing paths: connectivity to the local attacker machine (Kali Linux) and connectivity to the external internet (Google). Verifying this routing is essential for ensuring the environment is properly configured for the subsequent exploitation phases.

<p align="center">
  <img src=".assets/Metasploitable VM - Kali IP ping.png" alt="Ping to Attacker Machine" width="750"/>
  <br>
  <b>Figure 2: Validation of Outbound Connectivity to Attacker Machine</b>
</p>

<p align="center">
  <img src=".assets/Metasploitable VM - Google ping.png" alt="Ping to External Network" width="750"/>
  <br>
  <b>Figure 3: Validation of Outbound Connectivity to External Network</b>
</p>

---

## Phase 2: Reconnaissance & Surface Discovery

With network routing confirmed, the active attack chain initiates with comprehensive network and service enumeration to identify potential entry points on the target system.

### Service Enumeration
A full port scan was executed against the target using Nmap (`nmap -p- -sV 44.67.49.35`), revealing multiple exposed services including HTTP (Port 80), FTP (Port 21 running vsFTPd 2.3.4), SSH (Port 22), and Samba (Ports 139/445). 

<p align="center">
  <img src=".assets/Nmap full port scan.png" alt="Nmap Full Port Scan" width="750"/>
  <br>
  <b>Figure 4: Nmap Service Enumeration Results</b>
</p>

* **MITRE ATT&CK Mapping:** `T1046 - Network Service Discovery`

---

## Phase 3: Application Layer Exploitation (DVWA)

Following reconnaissance, the attacker first targets the web application layer (Damn Vulnerable Web App) running on port 80 to demonstrate data exfiltration and session hijacking.

### 1. SQL Injection (SQLi) & Data Extraction
Authentication bypass was initiated using the standard `' OR '1'='1` payload. To escalate the attack, a `UNION SELECT` statement was injected into the vulnerable input field. **As shown in Figure 5**, this command forced the database to reveal its internal schema, successfully enumerating hidden tables like `users` and `user_permissions`. 

Following this, a targeted extraction query was executed. **Figure 6 demonstrates the exact result of this command**, displaying the exfiltrated user accounts (e.g., admin, Gordon Brown) and their corresponding password hashes directly within the web application's interface.

<p align="center">
  <img src=".assets/SQL Injection - Table Enumeration.png" alt="SQL Injection Table Enumeration" width="800"/>
  <br>
  <b>Figure 5: UNION-based SQLi Database Enumeration</b>
</p>

<p align="center">
  <img src=".assets/SQL Injection - Data Extraction.png" alt="SQL Injection Data Extraction" width="800"/>
  <br>
  <b>Figure 6: Data Exfiltration via SQL Injection</b>
</p>

* **MITRE ATT&CK Mapping:** `T1190 - Exploit Public-Facing Application`

### 2. Cross-Site Scripting (XSS) & Session Hijacking
In parallel, a reflected Cross-Site Scripting (XSS) vulnerability was exploited by injecting a malicious JavaScript payload (`<script>alert(document.cookie)</script>`). **Figure 7 captures the execution of this payload**, which triggers a browser alert revealing the active `PHPSESSID` cookie. 

By intercepting the browser's web requests and manually swapping the current unauthenticated session cookie with the stolen `PHPSESSID`, the attacker bypassed the login screen entirely. **Figure 8 shows the exact result of this session hijacking**, granting immediate administrative access to the DVWA application without requiring a password.

<p align="center">
  <img src=".assets/XSS - Cookie Theft.png" alt="XSS Cookie Theft" width="800"/>
  <br>
  <b>Figure 7: XSS Payload Executing Cookie Theft</b>
</p>

<p align="center">
  <img src=".assets/XSS - Session Hijack .png" alt="Session Hijacking" width="800"/>
  <br>
  <b>Figure 8: Successful Session Impersonation via PHPSESSID</b>
</p>

* **MITRE ATT&CK Mapping:** `T1539 - Steal Session Cookie` / `T1550 - Use Alternate Authentication Material`

---

## Phase 4: System-Level Escalation & RCE

Having demonstrated application-layer compromise, the attacker escalates the attack to achieve Remote Code Execution (RCE) at the OS level using the vulnerable daemons discovered during the initial Nmap scan.

### vsFTPd & Samba Exploitation
Using Metasploit, the attacker targeted the vulnerable FTP service using the `exploit/unix/ftp/vsftpd_234_backdoor` module. Upon executing the exploit command, a backdoor command shell was successfully spawned. To verify the privilege level of the compromised session, the `whoami` command was run. **As evidenced in Figure 9, the command returned `root`, confirming absolute system takeover.**

Alternatively, the `exploit/multi/samba/usermap_script` module was deployed against the SMB service. **Figure 10 illustrates the successful execution of this exploit**, showing Metasploit establishing a secondary remote command execution (RCE) session directly into the target system via a vulnerable username map script.

<p align="center">
  <img src=".assets/Metasploitable - Root access proof.png" alt="Root Access via vsFTPd" width="750"/>
  <br>
  <b>Figure 9: Root Shell Achieved via vsFTPd Backdoor</b>
</p>

<p align="center">
  <img src=".assets/Metasploitable - Samba exploit .png" alt="Samba Exploitation" width="750"/>
  <br>
  <b>Figure 10: Alternative RCE via Samba Vulnerability</b>
</p>

* **MITRE ATT&CK Mapping:** `T1210 - Exploitation of Remote Services`

---

## Strategic Impact & Security Outcomes

* **Comprehensive Attack Surface Validation:** Demonstrated how attackers chain multiple vulnerabilities to systematically dismantle a target's defenses, moving from application-layer session hijacking to OS-level root compromise.
* **Identity and Session Abuse:** Proved that strong perimeter security can be bypassed if application-layer sessions (like PHP cookies) are not properly secured against client-side injection attacks.
* **Framework Mastery:** Successfully mapped the entire cyber kill chain to the industry-standard MITRE ATT&CK framework, proving a mature understanding of adversary tradecraft and systematic exploitation.
