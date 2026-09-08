---
title: Bruteforce Walkthrough - btlo
date: 2026-09-08
tags: ["linux", "logs", "basics"]
---

This is an walkthrough explaining how to complete the bruteforce chanllenge on Blue Team labs online
Challenge link: [bruteforce](https://blueteamlabs.online/home/challenge/bruteforce-16629bf9a2)

## Scenario

Can you analyze logs from an attempted RDP bruteforce attack?

One of our system administrators identified a large number of Audit Failure events in the Windows Security Event log.

There are a number of different ways to approach the analysis of these logs! Consider the suggested tools, but there are many others out there!

> NOTE: The given zip file needs to be opened with the password: `BTLO`

> The file can be downloaded directly via `curl -LO https://blueteamlabs.online/storage/files/00fd9853557296dd3312d4529c137f1cecb329d7.zip`

## Challenge Submission

**Question 1) How many Audit Failure events are there? (Format: Count of Events)**

We can use grep to check for `Audit Failure` to fileter the log file

```
grep -r "Audit Failure" BTLO_Bruteforce_Challenge.txt > failed.txt
```

Then we can use wc to check the total number of lines

```
remnux@remnux:~/btlo/bruteforce$ wc -l failed.txt 
3103 failed.txt
```

**Answer: 3103**

**Question 2) What is the username of the local account that is being targeted? (Format: Username)**

We can use grep to seperate out the `Account Name` to see each individual accounts

```
grep "Account Name" BTLO_Bruteforce_Challenge.txt > names.txt
```

Now we can use uniq to count how many times each account is meantioned

```
remnux@remnux:~/btlo/bruteforce$ sort names.txt | uniq -c
   3103 	Account Name:		-
      9 	Account Name:		BTLO
     13 	Account Name:		EC2AMAZ-UUEMPAU$
      8 	Account Name:		SYSTEM
   3103 	Account Name:		administrator
      4 	Network Account Name:	-
```

> NOTE: sort is needed because uniq can only detect duplicates if it is adjacent to each other

We can see that `administrator` account was targeted the most

**Answer: `administrator`**

**Question 3) What is the failure reason related to the Audit Failure logs? (Format: String)**

We can use grep again to search for `Failure Reason` in the log files

```
grep "Failure Reason" BTLO_Bruteforce_Challenge.txt > reason.txt
```

Then we can use uniq to check for duplicates

```
remnux@remnux:~/btlo/bruteforce$ sort reason.txt | uniq -c
   3103 	Failure Reason:		Unknown user name or bad password.
```

**Answer: `Unknown user name or bad password`**

**Question 4) What is the Windows Event ID associated with these logon failures? (Format: ID)**

At the very top of the log file we can see

```
Keywords	    Date and Time	        Source	                            Event ID  Task Category
Audit Failure	2/12/2022 7:22:00 AM	Microsoft-Windows-Security-Auditing	4625	  Logon	"An account failed to log on.
```

Under the `Event ID` we can see the answer

**Answer: 4625**

**Question 5) What is the source IP conducting this attack? (Format: X.X.X.X)**

Under the network information of the log, we can see `source network address`

**Answer: `113.161.192.227`**

**Question 6) What country is this IP address associated with? (Format: Country)**

We can use a online [geolocation](www.geolocation.com) to check which country the ip address
belongs to 

**Answer: `vietnam`**

**Question 7) What is the range of source ports that were used by the attacker to make these login requests? (LowestPort-HighestPort - Ex: 100-541)**

We first use grep to seperate the ports only

```
grep "Source Port" BTLO_Bruteforce_Challenge.txt > ports.txt
```

Then we can sort to ports according to their numbers

```
sort ports.txt > sorted_ports.txt
```

Then we can use head and tails respectively to get the lowest and highest ports

```
remnux@remnux:~/btlo/bruteforce$ head -10 sorted_ports.txt 
	Source Port:		-
	Source Port:		-
	Source Port:		-
	Source Port:		-
	Source Port:		49162
	Source Port:		49170
	Source Port:		49177
	Source Port:		49184
	Source Port:		49192
	Source Port:		49194
remnux@remnux:~/btlo/bruteforce$ tail -10 sorted_ports.txt 
	Source Port:		65483
	Source Port:		65488
	Source Port:		65496
	Source Port:		65497
	Source Port:		65508
	Source Port:		65515
	Source Port:		65516
	Source Port:		65526
	Source Port:		65529
	Source Port:		65534
```

**Answer: `49162-65534`**

Thank you and have a nice day

## See also

- [My projects](/projects) — tools I have built
- [Experience](/experience) — my background
