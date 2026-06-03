---
title: Sunset Walkthrough - Vulnhub
date: 2025-06-18
tags: ["linux", "ctf", "hacking"]
---

Sunset is crafted by the Whitecr0wz. This blog provides information on how to 
exploit the underlaying vulnerabilities in sunset machine

You can also easily download sunset through [vulnhub](https://www.vulnhub.com/entry/sunset-1,339/)

First we need to take a look at the available ports on the machine by port scanning using nmap

## Reconnaissance

`> nmap [ip addr]`

![nmap scan](/images/sunset/1.webp)

This show that the machine has 2 ports open, lets scan for more info

`> nmap [ip addr] -p 21,22 -A` 

![nmap scan](/images/sunset/2.webp)

This reveals more information about the target system. It shows that ftp allows for anonymous logins, Lets try that.

`> ftp [ip addr]`

![ftp](/images/sunset/3.png)

As you can see in the above image we were able to connect to the target system using ftp using anonymous credentials [anonymous:anonymous]

From there after typing “ls” we can see a file called “backup”, we can download this file using the get command

Now we should look inside the file “backup”

![backup](/images/sunset/4.png)

It appears to be hashes of different users, now copy paste the line which starts with sunset and enter it into a new file with a name of your choice. Then we will break the hash using john the ripper tool along with the rockyou.txt wordlist

## Password Cracking

`> john [filename] --wordlist=/usr/share/wordlists/rockyou.txt`

![john](/images/sunset/5.webp)

> Note: If you have already cracked the hash then in order for the password to appear again you have to enter `john — show sunset`

As you can see the password of user sunset is `cheer14`, we can use this credentials to login via ssh

`> ssh sunset@[ip addr]`

![ssh](/images/sunset/6.webp)

We can look around the files to find a file named `user.txt` which contains the user flag

![user](/images/sunset/7.webp)

Now we need to find the root flag which is probably located in the root directory

![root](/images/sunset/8.webp)

We cannot access the root folder access it doesn’t allow anyone to access it other than root itself, we need to find a way to escalate our privilege in order to access the files

## Privilege Escalation

`> sudo -l`

![root](/images/sunset/9.webp)

Running sudo -l as sunset show us all the binaries that we can run with sudo privileages. The binary “ed” can be run as sudo, we search in the website [GTFOBins](https://gtfobins.org/) to see what we can do with this binary

![gtfo](/images/sunset/10.webp)

Here we can see that under the sudo section that if it is run with sudo it doesn’t drop the privileges and we get a privilege escalation

`> sudo ed`

![root](/images/sunset/11.webp)

We are able to successfully spawn a shell with elevated privileges, now we can find the root flag also

![id](/images/sunset/12.webp)

And that is how you pawn the “sunset” machine, thank you and have a nice day.

## See also

- [Medium Story](https://medium.com/@pranavsuresh107/sunset-walkthrough-vulnhub-f13aad625c6f) - same story in medium
- [My projects](/projects) — tools I have built
- [Experience](/experience) — my background
