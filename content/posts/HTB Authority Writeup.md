+++
title = 'HTB Authority Writeup'
date = 2023-12-24T00:13:22-05:00
draft = false
+++

I'm back with another HackTheBox machine, and this is the first medium-difficulty machine I've made a writeup for: Authority.

## Enumeration

As always, we'll start out by adding the IP address to our `/etc/hosts` file and running an nmap scan.

![/etc/hosts](/images/authority/etc_hosts.png)
![nmap1](/images/authority/nmap1.png)
![nmap2](/images/authority/nmap2.png)

There are several open ports here, but let's start by looking at the web server on port 80.

![port 80](/images/authority/port80.png)

Looks like there isn't much going on here except for a default IIS page. Ports 139 and 445 being open suggests that there's an SMB server on the machine, so let's enumerate that with smbmap using a null session.

![smbmap null](/images/authority/smbmap_null.png)

## SMB 

There's a share called "Development" that we can read as a guest user. We'll look at this using smbclient.

![smbclient](/images/authority/smbclient.png)

So, Ansible services are running on the server. Let's grab all these files so we can analyze them on our Kali machine.

![mget](/images/authority/mget.png)

There are a lot of interesting files in here, but let's use `grep` to search for passwords. This gives a lot of results, so I'll only show the most interesting ones.

![grep password](/images/authority/grep_password.png)
![ansible creds](/images/authority/ansible_creds.png)
![tomcat creds](/images/authority/tomcat_creds.png)
![other creds](/images/authority/mainyml.png)
![ldap bind creds](/images/authority/ldapbind_creds.png)

We also find some password hashes in `PWM/defaults/`.

![hashes](/images/authority/pwm_hashes.png)

## Cracking Hashes

Following the steps from [this](https://www.bengrewell.com/cracking-ansible-vault-secrets-with-hashcat/) guide on cracking Ansible vault secrets, I was able to crack the `pwm_admin_password` hash using john. First, we need to get rid of the spaces (newlines are fine).

![pwm vault pass](/images/authority/pwm_admin_password.png)

Now that we've got the vault password we can use Ansible to decrypt the `pwm_admin_password`.

![pwm admin pass](/images/authority/pwm_admin_password2.png)

We can repeat this process for `ldap_admin_password`.

![ldap pass](/images/authority/ldap_pass.png)

It appears that the vault password was the same in both cases, so let's try it for `pwm_admin_login`.

![pwm login](/images/authority/pwm_login.png)

## Initial Foothold

Now, we've got a couple passwords and what appears to be a username, but no idea where to use them yet. Port 80 didn't have anything interesting, but let's check out the https-alt service running on port 8443.

![port 8443](/images/authority/port8443.png)

Looks like we've got a login page for a service called PWM. A quick google search gives us the [GitHub page](https://github.com/pwm-project/pwm) for this project, which is an open source password self-service application for LDAP directories. 

Let's try the pwm credentials we found.

![pwm login](/images/authority/login.png)
![error](/images/authority/error.png)

We got an error saying the directory is unavailable. What if we try this password in the configuration manager?

![config manager](/images/authority/config_manager.png)

It worked! Now we're logged into the configuration manager. It also appears that the same password gets us into the configuration editor.

![config editor](/images/authority/config_editor.png)

Since the error message earlier mentioned that the LDAP server is unavailable, maybe we can configure it to connect to our own LDAP server? To get an idea of how we can do this, let's download the configuration from the configuration manager and take a look at it.

Inside the configuration file, we find this interesting snippet:

![config](/images/authority/config.png)

It appears that the service is trying to connect to its own locally-hosted LDAP server, but it's either not defined correctly or the service isn't running.

Let's use `responder` to set up an LDAP server on our machine, then edit the configuration so that it connects to our LDAP server.

![responder](/images/authority/responder.png)

A bit of research shows that Responder hosts the LDAP server on port 389. Let's edit the config file to add our IP and port. We also can remove the 's' from `ldaps` since Responder isn't using a secure LDAP server.

![edit config file](/images/authority/edit_config.png)

We can now import the new configuration from the configuration manager. Then, go into the configuration editor and use the button shown below to initiate the connection.

![button](/images/authority/button.png)
![ldap clear](/images/authority/ldap_clear.png)

Looks like we got a cleartext password for the svc_ldap user! Let's try logging in using `evil-winrm`.

![user flag](/images/authority/user_flag.png)

Nice, we got the user flag!

## Privilege Escalation

Remember the files that we grabbed from the exposed SMB share? There was a folder called 'ADCS', which stands for Active Directory Certificate Services. This is a service that allows organizations to build their own public key infrastructure (PKI) and use all of the features that entails. You can learn more about it [here](https://learn.microsoft.com/en-us/previous-versions/windows/it-pro/windows-server-2012-r2-and-2012/hh831740(v=ws.11)). 

Misconfigured ADCS can often be leveraged by an attacker to get control of the entire Active Directory domain. Let's see if we can do something like that with this machine. We can upload [certify.exe](https://github.com/r3motecontrol/Ghostpack-CompiledBinaries/blob/master/Certify.exe) to find misconfigurations in ADCS. 

We'll upload the executable and then run it to find misconfigurations.

![upload](/images/authority/upload_certify.png)
![certify1](/images/authority/certify1.png)
![certify2](/images/authority/certify2.png)

So, there's a vulnerable template that we can abuse to get an administrator certificate. However, we need access to a computer on the domain to accomplish this. Let's see if we have the privileges to add a computer.

![get privs](/images/authority/get_priv.png)

It appears that we can add a computer to the domain using this account. I'll use `impacket-addcomputer` to add a computer that we can use to connect to the domain.

![impacket](/images/authority/impacket.png)

There's a great tool called [certipy](https://github.com/ly4k/Certipy) that we can use to enumerate ADCS and interact with the domain. As of the time that I'm writing this, there is some incompatibility with `certipy` and python3.11, but I found [this](https://github.com/secure-77/Certipy-Docker) docker version of `certipy` that runs it with python3.10.4. By the time you're reading this, the issue may be resolved.

Let's use `certipy` to get an administrator certificate.

![admin cert](/images/authority/admin_cert.png)

We can now use this certificate to get an "ldap shell" that has the same privileges as an administrator.

![ldap shell](/images/authority/ldap_shell.png)

Now, we just need to create a new user within the domain and add them to the "Domain Admins" group.

![add user](/images/authority/add_user.png)
![change pass](/images/authority/change_password.png)

We've created a user with admin privileges, and we can use `evil-winrm` again to connect to the machine as our new user.

![got root](/images/authority/got_root.png)

You can find the root flag on the Administrator's Desktop.

![root flag](/images/authority/root.png)

This machine required quite a bit of research, but it was pretty rewarding to root it in the end. I hope you enjoyed it as well.

Happy Hacking,

Dakota Vickers