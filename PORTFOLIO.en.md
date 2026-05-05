# Web to Active Directory Portfolio Brief

## Project Positioning

This project is a Web to Active Directory penetration testing lab. It simulates an attacker obtaining initial access from an external web service, then chaining credential exposure, credential reuse, lateral movement, and internal pivoting to ultimately validate Domain Admin compromise.

The goal is not to demonstrate a single isolated vulnerability. The project demonstrates the ability to:

- Evaluate how a web vulnerability can expand into internal enterprise risk
- Manually validate exploitability and practical impact
- Build and document a complete attack path
- Translate technical findings into weakness descriptions, root causes, impact, and remediation guidance
- Reproduce common real-world enterprise weakness combinations in a controlled lab

## One-minute Summary

This is a self-built Web to AD attack chain lab. The starting point is DVWA running on Ubuntu. Command Injection is used to obtain a reverse shell. After gaining shell access, an exposed `.env` credential is discovered on the web server. Because the service account is reused, it can authenticate to a dual-homed Windows workstation over SMB. The workstation is confirmed to have access to both the external and internal networks, making it a pivot point toward the internal Domain Controller. The attack chain then proceeds through credential extraction and offline hash cracking, ultimately validating Domain Admin access.

The project covers the full penetration testing flow: initial access, credential access, lateral movement, internal enumeration, privilege escalation, impact assessment, and remediation planning.

## Scope

| Role | System | IP |
| --- | --- | --- |
| Attacker | Kali Linux | 192.168.203.131 |
| Web Server | Ubuntu / DVWA | 192.168.203.130 |
| Pivot Host | Windows 10 dual-homed workstation | 192.168.203.132 / 192.168.88.110 |
| Domain Controller | Windows Server 2022 | 192.168.88.100 |

## Attack Path

```text
External Attacker
-> Command Injection on DVWA
-> Reverse Shell on Ubuntu
-> Exposed credential in .env
-> SMB login to Windows workstation
-> Pivot through dual-homed host
-> Domain Controller access
-> LSASS credential extraction
-> Hash cracking
-> Domain Admin compromise
```

## Technical Highlights

### 1. Web Exploitation

- Tested DVWA Command Injection behavior
- Used command separators to confirm OS command execution
- Established a reverse shell as the initial foothold

Evidence:

- `evidence/core/command-injection-request-response.png`
- `evidence/core/reverse-shell.png`

### 2. Sensitive Information and Credential Risk

- Discovered an exposed `.env` configuration file on the web server
- Obtained a service account credential
- Assessed the finding as a lateral movement risk, not just information disclosure

Evidence:

- `evidence/core/exposed-env-file.png`

### 3. Lateral Movement

- Tested SMB authentication using the obtained credential
- Successfully authenticated to a Windows workstation
- Confirmed excessive privileges and credential reuse risk

Evidence:

- `evidence/core/win10-smb-pwned.png`

### 4. Internal Pivot and Segmentation Weakness

- Remotely confirmed that the Windows workstation is dual-homed
- Identified it as a bridge between the external and internal networks
- Validated connectivity from the workstation to the Domain Controller

Evidence:

- `evidence/core/win10-ipconfig.png`
- `evidence/core/win10-ping-dc.png`

### 5. Active Directory Risk Expansion

- Performed internal service and Domain Controller validation
- Extracted credential material from LSASS
- Used hashcat for offline hash cracking
- Validated Domain Admin access

Evidence:

- `evidence/core/lsass-dump.png`
- `evidence/core/hashcat-cracking.png`
- `evidence/core/dc-smb-pwned.png`

## Capability Mapping

| Capability | How This Project Demonstrates It |
| --- | --- |
| Web security testing | Command Injection validation, payload testing, reverse shell |
| Linux fundamentals | Web directory enumeration, file permissions, configuration review |
| Windows / AD fundamentals | SMB authentication, domain accounts, Domain Controller, LSASS |
| Internal penetration testing | Lateral movement, pivot host analysis, external-to-internal pathing |
| Tool usage | nmap, crackmapexec, impacket, hashcat |
| Reporting | Attack path, evidence screenshots, impact explanation, remediation guidance |
| Risk understanding | Credential reuse, excessive privileges, dual-homed host, weak segmentation |

## Project Documents

- `README.md`: English project overview
- `README.zh-TW.md`: Traditional Chinese project overview
- `report/Report.en.md`: Full English report
- `report/Report.zh-TW.md`: Full Traditional Chinese report
- `evidence/core/`: Curated core evidence
- `evidence/raw/`: Raw testing screenshots and artifacts
