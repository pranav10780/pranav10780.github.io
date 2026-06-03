---
title: Investigating Windows — TryHackMe
date: 2025-07-12
tags: ["windows", "tryhackme", "blue team"]
---

This is a blue team based lab where you have to answer the questions given below in a windows machine

**1.Whats the version and year of the windows machine?**

We can open command prompt and run the command

`> systeminfo`

> NOTE: It may take a few seconds to show the results

![systeminfo](/images/windows/1.webp)

**Ans: Windows Server 2016**

**2.Which user logged in last?**

We can see this info using event viewer in windows. We need to select the security log under the windows logs section.

![logs](/images/windows/2.webp)

As you can see there are several different logs inside here, we need to filter only the login logs. We can easily do this by filtering for a specific event id

![evid](/images/windows/3.webp)
![evid](/images/windows/4.webp)
![evid](/images/windows/5.webp)

> NOTE: Since the machine was started today i will ignore the logs dated 2025 and instead focused on the 2019 logs

As you can see the last person to login to the computer (other than us today) was administrator

**ANS: Administrator**

**3.When did John log onto the system last?**

We can filter for John login by entering John in the find menu

![john](/images/windows/6.webp)
![john](/images/windows/7.webp)

**ANS: 03/02/2019 5:48:32 PM**

**4.What IP does the system connect to when it first starts?**

I think that we should check the startup apps to look for the answer inside the system information

![startup](/images/windows/8.webp)

**ANS: 10.34.2.3**

**5.What two accounts had administrative privileges (other than the Administrator user)?**

We can easily see this information in the computer managment

Computer Managment > Local User and Groups > Groups > Administrators> Properties

![admin](/images/windows/9.webp)

**ANS: Guest, Jenny**

**6.Whats the name of the scheduled task that is malicous.**

We can easily see this under the task sheduler

![task](/images/windows/10.webp)

We can see that this Clean file system is trying to run a script on port 1348 so it is malicious

**ANS: Clean file system**

**7.What file was the task trying to run daily?**

We have already got the answer to this question in the previous screen shot

**ANS: nc.ps1**

**8.What port did this file listen locally for?**

Again this answer can be found in the previous screen shot

**ANS: 1348**

**9.When did Jenny last logon?**

We can use the previous method of filtering for 4624(login event id) and then search for jenny

![evid](/images/windows/11.webp)

We get an error saying that there were no matches for jenny in the event viewer so we can conclude that she has never logged in the system

**ANS: Never**

**10.At what date did the compromise take place?**

We can safely assume this based on when jenny was created, we can see this in command prompt

`> net user Jenny`

![jenny](/images/windows/12.webp)

**ANS: 03/02/2019**

**11.During the compromise, at what time did Windows first assign special privileges to a new logon?**

See can see this using event viewer

![logon](/images/windows/13.webp)

We can filter for what happed on the day of compromise and specific event ids to determine when a privileage in elevated, user created, special priviileages to new logon, and a logon using explicit credentials

![privileage](/images/windows/14.webp)

We can see that on the specific log a lot of privileages are given

**ANS: 03/02/2019 4:04:49 PM**

**12.What tool was used to get Windows passwords?**

Since we already the saw a malicious file like nc.ps1 in the tmp folder lets check there

![tmp](/images/windows/15.webp)
![tmp](/images/windows/16.webp)

**ANS: Mimikatz**

**13.What was the attackers external control and command servers IP?**

Lets check the hosts file which acts as a local dns for the computer

![hosts](/images/windows/17.webp)

We can see that a dns poisoning has occured and the ip address of google.com has been changed

**Ans: 76.32.97.132**

**14.What was the extension name of the shell uploaded via the servers website?**

For the web server files we need to check the inetpub folder

![inet](/images/windows/18.webp)

We can the file extension

**ANS: .jsp**

**15.What was the last port the attacker opened?**

We can check inside the firewall settings to see the answer

![firewall](/images/windows/19.webp)

We can see a suspicious inbound rule called “Allow outside connections for development”, when we check the properties we get the answer

**ANS: 1337**

**16.Check for DNS poisoning, what site was targeted?**

We have already got the answer in the previous questions

**ANS: google.com**

## See also

- [Medium Story](https://medium.com/@pranavsuresh107/investigating-windows-tryhackme-febad3207308) - same story in medium
- [My projects](/projects) — tools I have built
- [Experience](/experience) — my background
