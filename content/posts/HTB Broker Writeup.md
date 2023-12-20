+++
title = 'HTB Broker Writeup'
date = 2023-12-17T12:05:26-05:00
draft = false
+++

We've got another HTB Writeup, this time it's an Easy-difficulty machine released earlier this year: Broker. This machine highlights a recently-disclosed vulnerability in a popular message broker service.

### Enumeration

First things first, I added the IP address to `/etc/hosts` and ran an nmap scan

![/etc/hosts](/images/broker/etc_hosts.png)
![nmap](/images/broker/nmap.png)

### Finding Initial Foothold

The scan found two open ports. When visiting the web app on port 80, we are prompted with an HTTP basic auth login

![basic auth](/images/broker/basic_auth.png)

Testing out some default credentials, I found that admin:admin works and gives us this page

![homepage](/images/broker/homepage.png)

Clicking on "Manage ActiveMQ broker" gives us some more information, including the ActiveMQ version: 5.15.15

![manage activeMQ](/images/broker/manage_activeMQ.png)

A quick google search for vulnerabilities in this version lead me to a CVE

![cve](/images/broker/cve.png)

This is a Remote Code Execution (RCE) vulnerability that has been used by several ransomware crews to attack ActiveMQ servers. Exploring the CVE page showed me that there's a Metasploit module available for this vulnerability, so I tried using that. 

The module defaults to using a Windows payload, but we know from the nmap scan that the machine is running Linux, so I had to change that. I also made sure that **LHOST** and **SRVHOST** were set to my IP address within the HackTheBox network. Here's what the module options looked like before running it

![metasploit options](/images/broker/metasploit_options.png)

Now we run the exploit to get a meterpreter shell on the target

![metasploit exploit](/images/broker/metasploit_exploit.png)

Finding the user flag from here was simple

![user flag](/images/broker/user_flag.png)

### Privilege Escalation

It's easy to discover that this user can run `/usr/sbin/nginx` as root without a password

![check sudo](/images/broker/check_sudo.png)

We can exploit this by creating a malicious nginx configuration that runs as root. The [nginx documentation](https://www.nginx.com/resources/wiki/start/topics/examples/full/) has an example configuration that we can go off of. You have to create an events section to specify the number of worker connections, so I just used the default value of 1024. I used the meterpreter session to create the config file at **tmp/privesc.conf** on the target machine

![config](/images/broker/config.png)

From here, I went back to the shell session, started the nginx server, and used curl to get the root flag

![root flag](/images/broker/root_flag.png)

&nbsp;  
Keep hacking,

Dakota Vickers