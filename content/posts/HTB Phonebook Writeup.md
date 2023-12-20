+++
title = 'HTB Phonebook Writeup'
date = 2023-11-24T00:54:07-05:00
draft = false
+++

Here's my first HackTheBox writeup: Phonebook. This is a retired web challenge that's pretty simple. Let's get started.

HackTheBox machines and challenges typically have names that hint at what their theme is. A name like 'Phonebook' definitely makes me think of brute-forcing. Maybe we'll come back to that later.

When we visit the target URL, we're greeted with a login page

![login page](/images/phonebook/login.png)

There's a message from someone named Reese, this might be a hint at a potential username. 

After trying to inject a lot of weird values, I found that wildcards were enabled. This suggests that the site is using LDAP for authentication. It also meant that I could login by entering '\*' in the username and password fields. This is a basic form of LDAP injection. Here's what we see after logging in with the wildcards

![search](/images/phonebook/search.png)

The search function is the only way to interact with this page, so I tried the wildcards again. They didn't seem to yield any interesting behavior with the search function, but the site does appear to be a "phonebook" of sorts.

![search 'a'](/images/phonebook/search1.png)

I also found an entry when searching for the name we found earlier

![reese search](/images/phonebook/reese.png)

If "reese" is a valid username, we can use the wildcard capability to brute-force the password. We can get the request's parameter names by proxying it through BurpSuite or looking at the source code

![burp](/images/phonebook/burp.png)

To brute-force the password, we need to first make a list of possible characters in the password. Then, we build the password one character at a time by iterating through the possible character list and appending a wildcard to it. I decided to use python to do this

![python code](/images/phonebook/brute.png)

It turns out that reese's password is the flag

![flag](/images/phonebook/get_password.png)

Flag: HTB{d1rectory_h4xx0r_is_k00l}

&nbsp;  
Until next time,

Dakota Vickers