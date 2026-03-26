# Full-Lifecycle Cyber Attack Simulation & MITRE ATT&CK Mapping

## Executive Summary
This project simulates a complete, end-to-end cyber attack lifecycle within an isolated virtualization lab. By executing a coordinated attack from a Kali Linux machine against a Metasploitable2 victim server, this exercise demonstrates the progression from initial reconnaissance to full system compromise. 

The project emphasizes realistic offensive attack chaining—moving from external service discovery, to OS-level root compromise, and finally to application-layer session hijacking—while mapping every tactical action to the industry-standard **MITRE ATT&CK Framework**. 

## Tech Stack & Frameworks
* **Offensive Tools:** Kali Linux, Metasploit Framework, Nmap, Burp Suite, Hydra 
* **Target Infrastructure:** Metasploitable2 (Linux), Damn Vulnerable Web App (DVWA), vsFTPd, Samba, VNC
* **Virtualization:** Nutanix AHV Cluster
* **Security Frameworks:** MITRE ATT&CK (T1046, T1190, T1539, T1210, T1021.002, T1110.001)

---

## System Architecture & Threat Model

The environment consists of an attacker machine and a vulnerable server operating on a shared, isolated Nutanix AHV subnet. 

<p align="center">
  <img src=".assets/Architecture%20Diagram.png" alt="Architecture Diagram and Attack Flow" width="850"/>
  <br>
  <b>Figure 1: Attack Architecture and Exploitation Flow</b>
</p>

### The Attack Flow:
The diagram above illustrates the relationship between the attacker and the exposed attack surface. The target machine (`44.67.49.35`) hosts several intentionally vulnerable services, allowing the simulation to progress systematically from external discovery to internal remote code execution and web application abuse.

---

## Phase 1: Environment Setup & Connectivity Verification

Before initiating the offensive attack chain, baseline network connectivity was established and verified. This ensures the isolated lab environment is routing traffic correctly, which is a critical prerequisite for advanced exploitation techniques like reverse shells.

### Network Validation
The Metasploitable2 victim machine was instructed to execute outbound `ping` commands. The first test verified local routing by successfully pinging the attacker's Kali Linux machine. This guarantees that if a payload is executed on the target, it has a valid network path to call back to the attacker.

<p align="center">
  <img src=".assets/Metasploitable%20VM%20-%20Kali%20IP%20ping.png" alt="Ping to Attacker Machine" width="750"/>
  <br>
  <b>Figure 2: Validation of Outbound Connectivity to Attacker Machine</b>
</p>

The second test verified external routing by successfully pinging Google's DNS servers. This confirms that the compromised machine could be used to exfiltrate data to the external internet or reach out to external Command and Control (C2) infrastructure.

<p align="center">
  <img src=".assets/Metasploitable%20VM%20-%20Google%20ping.png" alt="Ping to External Network" width="750"/>
  <br>
  <b>Figure 3: Validation of Outbound Connectivity to External Network</b>
</p>

---

## Phase 2: Reconnaissance & Surface Discovery

With network routing confirmed, the active attack chain initiates with comprehensive network and service enumeration to identify potential entry points on the target system.

### Service Enumeration
A full port scan was executed against the target using Nmap (`nmap -p- -sV 44.67.49.35`). The scan successfully mapped the attack surface, revealing multiple exposed services including HTTP (Port 80), FTP (Port 21 running vsFTPd 2.3.4), SSH (Port 22), and Samba (Ports 139/445). This enumeration directly dictated the exploitation strategy for Phase 3.

<p align="center">
  <img src=".assets/Nmap%20full%20port%20scan.png" alt="Nmap Full Port Scan" width="750"/>
  <br>
  <b>Figure 4: Nmap Service Enumeration Results</b>
</p>

* **MITRE ATT&CK Mapping:** `T1046 - Network Service Discovery`

---

## Phase 3: System-Level Compromise & RCE

Following reconnaissance, the attacker immediately targets the highly privileged, vulnerable OS-level daemons discovered during the Nmap scan to achieve Remote Code Execution (RCE) and root access.

### 1. VNC Authentication Bypass
The attacker first targeted the VNC service (port 5900). Using the `vncviewer` tool on the attacker machine, a direct connection attempt was made. The connection was established instantly without an authentication prompt, granting a remote desktop view of the server and exposing a critical security misconfiguration.

<p align="center">
  <img src=".assets/Metasploit%20-%20VNC%20viewer.png" alt="VNC Viewer Connection" width="750"/>
  <br>
  <b>Figure 5: Initial Unauthenticated VNC Connection Attempt</b>
</p>

To systematically validate this finding, Metasploit's `auxiliary/scanner/vnc/vnc_login` module was utilized. The scanner output explicitly confirmed the presence of a "blank" password state, allowing programmatic, unauthorized access to the VNC protocol.

<p align="center">
  <img src=".assets/Metasploitable%20-%20VNC%20exploit.png" alt="Metasploit VNC Login Scanner" width="750"/>
  <br>
  <b>Figure 6: VNC Login Vulnerability Verified via Metasploit</b>
</p>

* **MITRE ATT&CK Mapping:** `T1021.002 - Remote Services: VNC`

### 2. vsFTPd Exploitation & Root Escalation
Shifting focus to the FTP service, the attacker conducted a credential stuffing attack. Using the `hydra` network logon cracker, a dictionary attack was launched against the `msfadmin` account. The tool rapidly identified the valid credentials as `msfadmin:msfadmin`, highlighting a severe failure in the target's password complexity policy.

<p align="center">
  <img src=".assets/Metasploitable%20-%20vsFTPd%20password%20analysis.png" alt="Hydra FTP Password Analysis" width="800"/>
  <br>
  <b>Figure 7: Successful Dictionary Attack Against FTP Service</b>
</p>

Knowing the FTP daemon version (`vsFTPd 2.3.4`) was outdated, the attacker utilized Metasploit's `exploit/unix/ftp/vsftpd_234_backdoor` module. Executing this exploit successfully triggered the malicious backdoor embedded in that specific software version, establishing a listening command shell on port 6200.

<p align="center">
  <img src=".assets/Metasploitable%20-%20vsFTPd%20backdoor.png" alt="Metasploit vsFTPd Backdoor Exploit" width="750"/>
  <br>
  <b>Figure 8: Execution of the vsFTPd Backdoor Exploit</b>
</p>

With the shell established, the attacker needed to verify the current context and privilege level of the compromised session. Executing the `whoami` command returned `root`, verifying that the vsFTPd backdoor granted absolute, unrestricted administrative takeover of the operating system.

<p align="center">
  <img src=".assets/Metasploitable%20-%20Root%20access%20proof.png" alt="Root Access Confirmed" width="750"/>
  <br>
  <b>Figure 9: Confirmation of Root Shell Privileges via whoami</b>
</p>

### 3. Alternative Vector: Samba Exploitation
To demonstrate persistence and alternative entry points, the attacker targeted the SMB service. By deploying the `exploit/multi/samba/usermap_script` module, a secondary Remote Code Execution (RCE) session was successfully established directly into the target system, proving that the server was vulnerable from multiple independent vectors.

<p align="center">
  <img src=".assets/Metasploitable%20-%20Samba%20exploit%20.png" alt="Samba Usermap Exploit" width="750"/>
  <br>
  <b>Figure 10: Alternative RCE Achieved via Samba Vulnerability</b>
</p>

* **MITRE ATT&CK Mapping:** `T1110.001 - Brute Force: Credential Stuffing` / `T1210 - Exploitation of Remote Services`

---

## Phase 4: Application Layer Exploitation (DVWA)

Even with system-level compromise achieved, the attacker pivoted to exploit the web application layer (Damn Vulnerable Web App) on port 80. This demonstrates the ability to directly extract backend user data and hijack active web sessions.

### 1. SQL Injection (SQLi) & Data Extraction
Authentication bypass was initiated using a standard `' OR '1'='1` SQL injection payload. To escalate the attack, a `UNION SELECT` statement was injected into the vulnerable input field. This forced the database to reveal its internal architecture, successfully enumerating highly sensitive tables such as `users` and `user_permissions`.

<p align="center">
  <img src=".assets/SQL%20Injection%20-%20Table%20Enumeration.png" alt="SQL Injection Table Enumeration" width="800"/>
  <br>
  <b>Figure 11: UNION-based SQLi Database Enumeration</b>
</p>

Following the schema discovery, a targeted data extraction query was executed. This command forced the application to print the contents of the `users` table, successfully exfiltrating usernames (e.g., admin, Gordon Brown) and their corresponding password hashes directly to the attacker's screen.

<p align="center">
  <img src=".assets/SQL%20Injection%20-%20Data%20Extraction.png" alt="SQL Injection Data Extraction" width="800"/>
  <br>
  <b>Figure 12: Data Exfiltration via SQL Injection</b>
</p>

* **MITRE ATT&CK Mapping:** `T1190 - Exploit Public-Facing Application`

### 2. Cross-Site Scripting (XSS) & Session Hijacking
In parallel, the attacker exploited a reflected Cross-Site Scripting (XSS) vulnerability. By injecting a malicious JavaScript payload (`<script>alert(document.cookie)</script>`), the application executed the script within the browser's context. This triggered a pop-up alert that successfully exposed the victim's active `PHPSESSID` session cookie.

<p align="center">
  <img src=".assets/XSS%20-%20Cookie%20Theft.png" alt="XSS Cookie Theft" width="800"/>
  <br>
  <b>Figure 13: XSS Payload Executing Cookie Theft</b>
</p>

Armed with the stolen `PHPSESSID`, the attacker intercepted their own browser requests and manually injected the compromised cookie. This technique bypassed the login screen entirely, forcing the server to recognize the attacker as an authenticated user and granting immediate administrative access to the DVWA portal.

<p align="center">
  <img src=".assets/XSS%20-%20Session%20Hijack%20.png" alt="Session Hijacking" width="800"/>
  <br>
  <b>Figure 14: Successful Session Impersonation via PHPSESSID</b>
</p>

* **MITRE ATT&CK Mapping:** `T1539 - Steal Session Cookie` / `T1550 - Use Alternate Authentication Material`

---

## Strategic Impact & Security Outcomes

* **Comprehensive Attack Surface Validation:** Demonstrated how attackers chain multiple vulnerabilities to systematically dismantle a target's defenses, navigating from exposed misconfigurations to OS-level root compromise and application-layer session abuse.
* **Identity and Session Abuse:** Proved that strong perimeter security can be bypassed if application-layer sessions (like PHP cookies) are not properly secured against client-side injection attacks.
* **Framework Mastery:** Successfully mapped the entire cyber kill chain to the industry-standard MITRE ATT&CK framework, proving a mature understanding of adversary tradecraft and systematic exploitation.
