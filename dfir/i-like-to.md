In this HTB "Sherlock"-challenge, we assess a system that's been attacked. We need to answer some questions by navigation through log files and as such get an overview of how the system was attacked.

In this case, we find an **`ASPX`-webshell**, created in **Ruby**.

1. Name of the ASPX webshell uploaded by the attacker?
	1. `move.aspx`
2. What was the attacker's IP address?
	1. `10.255.254.3`
3. What user agent was used to perform the initial attack?
	1. `Ruby`
4. When was the ASPX webshell uploaded by the attacker?
	1. `2023-07-12 11:24:30`
5. The attacker uploaded an ASP webshell which didn't work, what is its filesize in bytes?
	1. `1362`
6. Which tool did the attacker use to initially enumerate the vulnerable server?
	1. `nmap`
7. We suspect the attacker may have changed the password for our service account. Please confirm the time this occurred (UTC)
	1. `2023-07-12 11:09:27`
8. What was the useragent that the attacker used to access the webshell?
	1. `Mozilla/5.0+(X11;+Linux+x86_64;+rv:102.0)+Gecko/20100101+Firefox/102.0`
