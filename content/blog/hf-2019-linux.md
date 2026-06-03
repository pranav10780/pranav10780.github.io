---
title: HF 2019-Linux
date: 2025-06-18
tags: ["linux", "ctf", "hacking"]
---

We are going to try to break into HF 2019-Linux machine which can be downloaded [here](https://www.vulnhub.com/entry/hacker-fest-2019,378/)

Let start off by basic scanning using nmap

## Reconnaissance

`> nmap [ip addr]`

![nmap](/images/hf2019/1.webp)

We can see that there is 4 ports available in this machine, now lets take a closer look at the ports

`> nmap [ip addr] -p 21,22,80,10000 -A`

![nmap_advanced](/images/hf2019/2.webp)
![nmap_advanced](/images/hf2019/3.webp)

We can see that there are a lot of files in the ftp server available for us, in order to download all the files available in the server we can use wget

`> wget -m ftp://anonymous:anonymous@[ip addr]`

This should save every file inthe server to a neat folder called [ip addr]

![files](/images/hf2019/4.webp)

Now most of the files are actually of little to no interest or use to us at all, usually the wp-config.php file usually contains senstivite information but in this case it is useless

![wp-config](/images/hf2019/5.webp)

At this point the ftp server is a dead end so now we can explore other ports such as port 80 which hosts a webserver using wordpress. Wordpress is a jackpot for vulnerabilities due to outdated pulgins etc

![wordpress](/images/hf2019/6.webp)

We can see that a blog site is hosted on this webserver lets enumerate the site using `wpscan`

`> wpscan --url http://[ip addr]`

![wpscan](/images/hf2019/7.png)
![wpscan](/images/hf2019/8.png)
![wpscan](/images/hf2019/9.webp)

We get a lot of data thorugh this enumeration scanning, after a quick search through all the data we find an interesting pulgin with a sql injection vulnerability

## Exploit

`> searchsploit wp google map`

![wp](/images/hf2019/10.webp)

Using msfconsole with this module we can run this module

`> msfconsole`

![msfconsole](/images/hf2019/11.png)
![msfconsole](/images/hf2019/12.webp)

> Note: always remember to set rhosts before running any exploit

After setting all the neccessary options such as rhosts to target machine ip address we can run it

![exploit](/images/hf2019/13.webp)

> Note: run and exploit do the same thing inside metasploit

We can see in the above picture that we get a username and hash, now we can crack it using john the ripper. Copy the username and password and paste it into a file in the following format `[username:password]`

## Password Cracking

`> john [filename] --wordlist=/usr/share/wordlists/rockyou.txt`

![john](/images/hf2019/14.png)

We get that the password of user webmaster is `kittykat1` using this credentials we can login via ssh

`> ssh webmaster@[ip addr]`

![ssh](/images/hf2019/15.webp)

We can get the user flag which is present the user directory

![flag](/images/hf2019/16.webp)

In order to find the root flag we must change our user to root

`> sudo -l`

![sudo](/images/hf2019/17.webp)

By running `sudo -l` we can see that we have permission to run any command as sudo, thus we can simply change to root user by `sudo su`

`> sudo su`

![su](/images/hf2019/18.webp)

After changing to root user, by typing cd we go to the home directory of the user `/root` where we can find the root flag

Thank you and have a nice day.

## See also

- [Medium Story](https://medium.com/@pranavsuresh107/hf-2019-linux-a1d5cb3a0cb5) - same story in medium
- [My projects](/projects) — tools I have built
- [Experience](/experience) — my background
