+++
title = 'eJPT Review'
date = 2023-12-01T23:46:46-05:00
draft = false
+++

Having attained my CEH (Certified Ethical Hacker) last year, I wanted to take a practical certification exam that required me to actually carry out some form of penetration testing in order to pass. I wasn't very interested in the CEH Practical exam, and I didn't feel confident enough in my skills to tackle the OSCP (Offensive Security Certified Professional). The eLearnSecurity certifications have become more and more popular over the last few years, and the eJPT (eLearnSecurity Junior Penetration Tester) certification seemed to be what I was looking for.

For some background, I have a bachelor's degree in Computer Science, and I've been working in information security for just under three years. In my current role, I am an application penetration tester with a primary focus on web applications and APIs. I feel that I've been diligent about keeping my skills honed as time has gone by, but I have been passive in my efforts to acquire new certifications since I passed the CEH exam. I don't necessarily think that certifications are a requirement to be successful in this industry, but they serve as proof to other infosec professionals of one's skills and knowledge in a specific area. For me, it also can serve as a reassurance that I *am* actually qualified to work in this industry, as I'm someone who has often dealt with imposter syndrome. In late October of this year, I decided the eJPT would be a way for me to put my nose to the grindstone and begin making progress towards getting myself some more certifications.

As far as certifications go, the eJPT is very affordable. INE, the company that acquired eLearnSecurity in 2019, offers their "Fundamentals" subscription for just $39/month or $299/year. The "Penetration Testing Student" course, which covers everything you need to know for the eJPT exam, is just under 150 hours of material. I finished the training within a few weeks, which is honestly very doable if you already have some experience with the topics being covered and watch the training videos on 1.5x speed. The exam voucher itself costs $200 and includes a free retake if needed.

## The Course Material

This course covers a lot, OSINT (Open Source Intelligence), enumeration, networking basics, pentesting methodology (information gathering, footprinting and scanning, enumeration, vulnerability scanning, exploitation, and post-exploitation), and common tools used by security professionals. There's a huge focus on the Metasploit Framework and how it can be used to carry out each step of the methodology, and I think this content is one of the best aspects of the training. I already had a good bit of experience with Metasploit, but I learned a lot about the differences between using it in a professional context versus using it for something like a CTF or HackTheBox.

I want to point out that this course is primarily focused on attacking network services and escalating your privileges from there. There is some basic material on exploiting web applications, but only enough to have a basic understanding of how to get a foothold on a machine. 

The course also has an interesting focus on pivoting. For those that don't know, pivoting is a technique where you get access to a machine that is publicly exposed but also part of a private network. You then "pivot" off of that machine to mount attacks against other machines on the private network. This is a concept I knew in theory but had never really put into practice until doing this course. I really like that they took the time to focus on this, especially because a few of the exam questions require knowing how to pivot.

As with any certification training, it's imperative to take good notes. Not only will this help you immensely with the exam itself, but you'll learn a lot of techniques that can be useful down the line in a real penetration test. Having a well-organized set of reference material can make it easier and faster for you to perform tasks that you've already done before.

I also recommend not looking at the solutions when completing the labs unless you have spent a good amount of time trying to solve it on your own. Between the learning materials and Google (which will become your best friend), you should be able to solve these labs on your own. If you do get stuck and have to peek at the solution, make sure you go back later and try to do the lab again on your own.

## The Exam

I had some experience with TryHackMe and HackTheBox, so I didn't do much preparation for the exam other than completing the course. On the Friday I completed the training, I purchased my exam voucher, planning to take the exam that Sunday.

The exam is 48 hours, though most people who complete the training won't need even half that time to pass. It consists of a number of multiple choice questions (mine had 35 questions but I don't know if this is standard with each exam), but the answers to these questions can only be found by enumerating and exploiting a number of vulnerable machines. 

The questions are mostly things like "What kind of vulnerability did you exploit to get access to machine 1?" or "What vulnerable service did you attack to get access to machine 2?". There were also some CTF-like questions where a flag had to be obtained from the root directory of a machine or a specific user's password hash had to be cracked. You can view all of the questions from the very start of the exam, so you can use these questions when you're stuck to get an idea of where you should be looking.

You're given a fictional scenario of being a junior penetration tester working on your first solo pentest for a client. You have access to an in-browser virtual machine that has all of the tools you'll need to complete the exam. The VM is part of the "client's" DMZ (Demilitarized Zone), which contains a number of machines on it (this number varies with each exam, mine had 6). One of these machines is part of an internal network that has a number of other machines on it (mine had 2 others), but you won't know which machine this is until you have access to it.

Once I scanned the DMZ network and discovered all of the hosts, I began enumerating the open ports and services on each machine. It's important to take good notes at this stage because you're going to reference the information you find a **lot** throughout the rest of the exam. Nmap is your friend in this step, but I also made use of some Metasploit auxiliary modules.

From here, everyone's experience is going to differ quite a bit, so I won't go into too much detail about how I compromised each machine. Of the 6 other machines in the DMZ, only 4 of them were vulnerable (as far as I know), one of which was the machine that I needed to pivot off of in order to gain access to the private network.

On two of the machines, I was able to exploit misconfigurations in their web applications to upload a reverse shell and get initial access. None of the exam questions required me to have root access to these two machines, so that's where I stopped for these.

The third machine was using a vulnerable version of a popular CMS for it's web application, and I was able to easily get root access to this machine using a Metasploit module.

The fourth machine was the one I struggled with the most in terms of getting initial access. There was another web application on this one, but after I spent a lot of time digging into it, it turned out to be a red herring. I eventually got access by brute-forcing the Administrator's SMB password. Since I already had access to the three other machines and they didn't appear to be on any private networks, I had already figured that this would be my pivot machine. Once I had access to this machine, I was able to verify that it was, in fact, part of a private network. From there, I began enumerating the other machines on the network. I didn't actually need to exploit these other machines to answer any of the questions.

After about 8 hours of actual work (I took a few breaks during the exam), I had answered 33 of the 35 questions. The last two questions involved cracking two hashes that I had obtained, but it was late, so I set my hash crackers to run overnight and went to bed. The next day was a Monday and my tools hadn't found anything so I decided to run them with a larger wordlist and come back to it after the workday. They still hadn't cracked the hashes after work, and I only needed 70% to pass the exam, so I decided to submit my answers without completing those two questions. It turns out I had nothing to worry about because those were the only two questions I missed!

## Conclusion

I honestly thought the eJPT exam was fairly easy overall, but still very fun and challenging at times. Just because it's on the easier side doesn't mean I think it isn't worth doing. The eJPT is excellent for budding penetration testers who want to get their first practical certification, or for more experienced pentesters who want a refresher on some of the basics. For me, the ease with which I completed the exam was a much-needed confidence boost. I now think that I'm more than ready to begin training for the OSCP, which I plan to start on after the new year.

If you're on the fence about starting the eJPT certification journey, especially if you're new to penetration testing or still early on in your career, I highly recommend that you just go for it. It's affordable, you'll learn a lot, and it just feels good to pass a practical certification exam, proving to yourself and to others that you've got what it takes.

![eJPT Certificate](/images/ejpt.PNG)

Thanks for reading,

Dakota Vickers