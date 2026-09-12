---
title: Cascade - BTLO
date: 2026-09-12
tags: ["linux", "btlo", "writeup"]
---

## Investigation Submission

**1) What is the hostname of the affected machine? Who is the registered owner?**

We need to check the `C:\Users\BTLOTest\Desktop\Artefacts\DevEvidence\Triage\UserInfo\whoami.txt`
or if via wsl then the files is located at `/mnt/c/Users/BTLOTest/Desktop/Artefacts/DevEvidence/TriageFiles/UserInfo/whoami.txt`

```
btlo@HammerInTheVault:~$ cat /mnt/c/Users/BTLOTest/Desktop/Artefacts/DevEvidence/TriageFiles/UserInfo/whoami.txtDESKTOP-T4SU469\Travis Bruce
```

**Answer: `DESKTOP-T4SU469\Travis Bruce`**

**2) During the analysis of running processes, a suspicious process was discovered that is likely to represent the initial execution. What is the name and pid of this process? **

Yeah so not gonna lie this was way harder than i expected, i wasted lots of time analysing log files and i eventually understood that i was supposed to use volatility3 on the ubuntu wsl to read the dmp file.

Currently via volitility i have managed to figure out an cmd process is calling a `killprocess.bat` from the bitrock install dir but it seems the parent process of the cmd has already exited.

```
btlo@HammerInTheVault:~/tools/volatility3$ python3 vol.py -f /mnt/c/Users/BTLOTest/Desktop/Artefacts/DevEvidence/case001-travisbruce.dmp windows.pstree > pstree.txt
btlo@HammerInTheVault:~/tools/volatility3$ python3 vol.py -f /mnt/c/Users/BTLOTest/Desktop/Artefacts/DevEvidence/case001-travisbruce.dmp windows.cmdline > cmdline.txt

btlo@HammerInTheVault:~/tools/volatility3$ grep -n "6352" pstree.txt cmdline.txt
pstree.txt:201:3188     6352    cmd.exe 0xdd0af16e1080  1       -       3       False   2026-02-23 09:30:53.000000 UTC N/A                                                                                                                  \Device\HarddiskVolume2\Windows\System32\cmd.exe C:\Windows\system32\cmd.exe  /K call  "@@BITROCK_INSTALLDIR@@\killprocess.bat" "httpd.exe"                                                                                                  C:\Windows\system32\cmd.exe
```

As you can see we now need to find the process which has pid 6352 which is the parent processo of 3188 cmd

```
btlo@HammerInTheVault:~/tools/volatility3$ python3 vol.py -f /mnt/c/Users/BTLOTest/Desktop/Artefacts/DevEvidence/case001-travisbruce.dmp windows.psscan > psscan.txt

btlo@HammerInTheVault:~/tools/volatility3$ cat psscan.txt | grep 63523188    6352    cmd.exe 0xdd0af16e1080  1       -       3       False   2026-02-23 09:30:53.000000 UTC  N/A     Disabled
```

We are unfourtunately not able to find the process associated to 6352, now lets check the logs again.

```
btlo@hammerinthevault:/mnt/c/Users/BTLOTest/Desktop/Artefacts/DevEvidence/TriageFiles/BasicInfo$ jobs[1]+  Stopped                 grep --color=auto -n "6352" > 6352.txt
btlo@HammerInTheVault:/mnt/c/Users/BTLOTest/Desktop/Artefacts/DevEvidence/TriageFiles/BasicInfo$ cat 6352.txt
```
As u can see i was mistaked and followed the wrong lead hope you can learn from this, back to the logs we gooooo.

The logs went nowhere so i decided to check the memory dumps again.

```
btlo@hammerinthevault:~/tools/volatility3$ grep -in "Public\\\\Music\|Public\\\\Videos\|Public\\\\Pictures\|\\\\Temp\\\\\|\\\\AppData\\\\Local\\\\Temp" pstree.txt cmdline.txt
pstree.txt:79:***** 12596       2836    _uninstall2836  0xdd0af7e14340  0       -       3       False   2026-02-23 08:57:45.000000 UTC                                                                                                      2026-02-24 07:42:19.000000 UTC   \Device\HarddiskVolume2\Users\TRAVIS~1\AppData\Local\Temp\_uninstall931BB399\_uninstall2836.000                                                                                                             --
pstree.txt:223:5080     3332    activesyncx86.  0xdd0af0b8c2c0  0       -       3       False   2026-02-26 11:03:30.000000 UTC                                                                                                              2026-02-26 11:20:45.000000 UTC   \Device\HarddiskVolume2\Users\Public\Music\activesyncx86.exe    -       -
pstree.txt:226:*** 3664 896     sdel.exe        0xdd0af39dc300  0       -       3       False   2026-02-26 11:21:42.000000 UTC                                                                                                              2026-02-26 11:21:42.000000 UTC   \Device\HarddiskVolume2\Users\Public\Music\sdel.exe     -       -
```

We can see that `activesyncx` is a microsoft syncronisation protocol but its file location is in Music directory which is abnormal so it is trying to hide as a legitimate microsoft service, the process `sdel.exe` is also run from the same directory, coincidence, I THINK NOT.

**Answer: `activesyncx86.exe:5080`**

**3) From which root parent image was it triggered?**

Before we try to find the parent process fo `activesyncx86` i would like to map out to process tree of `sdel.exe` and find the connection between them.

```
btlo@HammerInTheVault:~/tools/volatility3$ grep -n -A10 "activesyncx86.exe" pstree.txt
223:5080        3332    activesyncx86.  0xdd0af0b8c2c0  0       -       3       False   2026-02-26 11:03:30.000000 UTC 2026-02-26 11:20:45.000000 UTC                                                                                       \Device\HarddiskVolume2\Users\Public\Music\activesyncx86.exe     -       -
224-* 9912      5080    dllhost.exe     0xdd0af3341080  24      -       3       False   2026-02-26 11:14:14.000000 UTC N/A                                                                                                                  \Device\HarddiskVolume2\Windows\System32\dllhost.exe     C:\Windows\System32\dllhost.exe C:\Windows\System32\dllhost.exe
225-** 896      9912    cmd.exe 0xdd0af1bf3340  0       -       3       False   2026-02-26 11:21:42.000000 UTC  2026-02-26 11:21:42.000000 UTC                                                                                              \Device\HarddiskVolume2\Windows\System32\cmd.exe -       -
226-*** 3664    896     sdel.exe        0xdd0af39dc300  0       -       3       False   2026-02-26 11:21:42.000000 UTC 2026-02-26 11:21:42.000000 UTC                                                                                       \Device\HarddiskVolume2\Users\Public\Music\sdel.exe      -       -
--snip--
```

As you can see from the above snippet the process tree goes something like this:

```
UNKNOWN (3332) -> activesyncx86.exe (5080) -> dllhost.exe (9912) -> cmd.exe (896) -> sdel.exe (3664)
```

Now we get the connection between them, now we need to find this unknown program to get the answer.

We first check the pstree.txt for any references

```
btlo@HammerInTheVault:~/tools/volatility3$ grep "3332" pstree.txt  -n223:5080        3332    activesyncx86.  0xdd0af0b8c2c0  0       -       3       False   2026-02-26 11:03:30.000000 UTC 2026-02-26 11:20:45.000000 UTC                                                                                       \Device\HarddiskVolume2\Users\Public\Music\activesyncx86.exe     -       -
```

Since the pstree is a dead end lets try scanning the logs.

```
--snip--
/mnt/c/Users/BTLOTest/Desktop/Artefacts/DevEvidence/TriageFiles/BasicInfo/Hashes_sha256_System32_AllFiles_and_Dates.txt:11819:52d43dde550b14152028e3baaa4294d2d3b95bff1e62f4e3332cb211d5451931 2026:02:02:17:06:38  C:\Windows\system32\trie.dll
/mnt/c/Users/BTLOTest/Desktop/Artefacts/DevEvidence/TriageFiles/BasicInfo/Hashes_sha256_System32_AllFiles_and_Dates.txt:13026:8de032ad5f2e5efb4d6a872f0262298df6c83d9be8dcc3332e45b889e8066cea 2026:02:02:17:07:26  C:\Windows\system32\Windows.UI.Input.Inking.dll
/mnt/c/Users/BTLOTest/Desktop/Artefacts/DevEvidence/TriageFiles/BasicInfo/PsLoglist.txt:9799:[3332] Service Control Manager
```

We can see a `3332` but it's not a pid but still its maybe worth checking out?

```
btlo@HammerInTheVault:~/tools/volatility3$ grep -n -A10 -B4 "3332" /mnt/c/Users/BTLOTest/Desktop/Artefacts/DevEvidence/TriageFiles/BasicInfo/PsLoglist.txt
--snip--
9799:[3332] Service Control Manager
9800-   Type:     INFORMATION
9801-   Computer: DESKTOP-T4SU469
9802-   Time:     2/22/2026 6:43:23 AM   ID:       7040
9803-   User:     NT AUTHORITY\SYSTEM
9804-The start type of the Background Intelligent Transfer Service service was changed from auto start to demand start.
--snip--
```

It was just Event ID 3332, just a conincedence, back to logs we gooo.

THIS WRITEUP IS A WORK IN PROGRESS AND I HAVE NOTE GOT ALL THE ANSWERS
