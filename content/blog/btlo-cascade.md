---
title: Cascade - btlo
date: 2026-09-18
tags: ["linux", "btlo", "writeup"]
---

## Investigation Submission

**1) What is the hostname of the affected machine? Who is the registered owner?**

We need to check the `C:\Users\BTLOTest\Desktop\Artefacts\DevEvidence\Triage\UserInfo\whoami.txt`
or if via wsl then the files is located at `/mnt/c/Users/BTLOTest/Desktop/Artefacts/DevEvidence/TriageFiles/UserInfo/whoami.txt`

```
btlo@HammerInTheVault:~$ cat /mnt/c/Users/BTLOTest/Desktop/Artefacts/DevEvidence/TriageFiles/UserInfo/whoami.txt
DESKTOP-T4SU469\Travis Bruce
```

**Answer: `DESKTOP-T4SU469\Travis Bruce`**

**2) During the analysis of running processes, a suspicious process was discovered that is likely to represent the initial execution. What is the name and pid of this process?**

Yeah so not gonna lie this was way harder than i expected, i wasted lots of time analysing log files and eventually stumbled upon this in the memory dump `case001-travisbruce.dmp`.

```
btlo@hammerinthevault:~/tools/volatility3$ grep -in "Public\\\\Music\|Public\\\\Videos\|Public\\\\Pictures\|\\\\Temp\\\\\|\\\\AppData\\\\Local\\\\Temp" pstree.txt cmdline.txt
pstree.txt:79:***** 12596   2836    _uninstall2836  0xdd0af7e14340  0       -       3       False   2026-02-23 08:57:45.000000 UTC   2026-02-24 07:42:19.000000 UTC   \Device\HarddiskVolume2\Users\TRAVIS~1\AppData\Local\Temp\_uninstall931BB399\_uninstall2836.000   --
pstree.txt:223:     5080    3332    activesyncx86.  0xdd0af0b8c2c0  0       -       3       False   2026-02-26 11:03:30.000000 UTC   2026-02-26 11:20:45.000000 UTC   \Device\HarddiskVolume2\Users\Public\Music\activesyncx86.exe    -       -
pstree.txt:226:***  3664    896     sdel.exe        0xdd0af39dc300  0       -       3       False   2026-02-26 11:21:42.000000 UTC   2026-02-26 11:21:42.000000 UTC   \Device\HarddiskVolume2\Users\Public\Music\sdel.exe     -       -
```

>NOTE: `pstree.txt` and `cmdline.txt` are just volatilty3 plugins `windows.pstree` and `windows.cmdline` respectively i transfered the output so i don't have to run the command multiple times.
>
>`python3 vol.py -f /mnt/c/Users/BTLOTest/Desktop/Artefacts/DevEvidence/case001-travisbruce.dmp windows.pslist > pslist.txt`
>
>`python3 vol.py -f /mnt/c/Users/BTLOTest/Desktop/Artefacts/DevEvidence/case001-travisbruce.dmp windows.cmdline > cmdline.txt`

We can see that `activesyncx` is a microsoft syncronisation protocol but its file location is in Music directory which is abnormal so it is trying to hide as a legitimate microsoft service, the process `sdel.exe` is also run from the same directory, ***coincidence, I THINK NOT.***

**Answer: `activesyncx86.exe:5080`**

**3) From which root parent image was it triggered?**

Before we try to find the parent process for `activesyncx86` i would like to map out to process tree of `sdel.exe` and find the connection between them.

```
btlo@HammerInTheVault:~/tools/volatility3$ grep -n -A10 "activesyncx86.exe" pstree.txt
223:5080        3332    activesyncx86.  0xdd0af0b8c2c0  0       -       3       False   2026-02-26 11:03:30.000000 UTC 2026-02-26 11:20:45.000000 UTC   \Device\HarddiskVolume2\Users\Public\Music\activesyncx86.exe     -       -
224-* 9912      5080    dllhost.exe     0xdd0af3341080  24      -       3       False   2026-02-26 11:14:14.000000 UTC N/A                              \Device\HarddiskVolume2\Windows\System32\dllhost.exe     C:\Windows\System32\dllhost.exe C:\Windows\System32\dllhost.exe
225-** 896      9912    cmd.exe 0xdd0af1bf3340  0       -       3       False   2026-02-26 11:21:42.000000 UTC  2026-02-26 11:21:42.000000 UTC          \Device\HarddiskVolume2\Windows\System32\cmd.exe -       -
226-*** 3664    896     sdel.exe        0xdd0af39dc300  0       -       3       False   2026-02-26 11:21:42.000000 UTC 2026-02-26 11:21:42.000000 UTC   \Device\HarddiskVolume2\Users\Public\Music\sdel.exe      -       -
--snip--
```

As you can see from the above snippet the process tree goes something like this:

```
UNKNOWN (3332) -> activesyncx86.exe (5080) -> dllhost.exe (9912) -> cmd.exe (896) -> sdel.exe (3664)
```

Unfortunately this is all we can find from the memory dump, but there is something far more precious hiding in the kaperesults.

```
btlo@HammerInTheVault:/mnt/c/Users/BTLOTest/Desktop/Artefacts/DevEvidence/kaperesults/out$ ls
2026-03-02T144216_InfectedTriage.zip  2026-03-02T14_42_16_1868267_ConsoleLog.txt
```

After extracting the `2026-03-02T144216_InfectedTriage.zip` we will get a hard disk image file we contains a lot of information but we are specifically looking for sysmon logs as it records details system activity which may not be record by the previous memory dump.

We can serach for pid `3332` in the `D:\C\Windows\System32\winevt\logs\Microsoft-Windows-Sysmon%4Operational.evtx` log using the filter to search between 2/26/2026 11:00:00 AM to 2/26/2026 11:59:00 AM and specifically filter form event id 1 to check the creation of a new process and then use the find action to search for `3332`.

![@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@](/images/cascade/1948.png)

As we can see in the above image that the pid `3332` is cmd.exe and it is parent process is vscode so the process tree is now like this.

```
code.exe (1948) -> cmd.exe (3332) -> activesyncx86.exe (5080) -> dllhost.exe (9912) -> cmd.exe (896) -> sdel.exe (3664)
```

>FUNFACT: the atacker forgot to change the metadata of `activesyncx86.exe` which is actually `apollo.exe`

**Answer: `1948:Code.exe (Microsoft VS Code)`**

**4) What file was downloaded after the installation was triggered? What is the server that hosted this file?**

We can use the newly found pid `1948` to search in the sysmon logs.

![@@@@@@@@@@@@@@@@@@@@@@@@@@@](/images/cascade/curl.png)

We can see that curl was used to download the `XdaWgasSWf.exe` as `activesync-updater.exe`.

**Answer: `C:\Users\Public\Music\XdaWgasSWf.exe, http://167.172.74.103/sync/data/activesync-updater.exe`**

**5) Which installation triggered the initial access on the dev workstation?**

Looking around the logs near the time stamp of 11:03:27.944, we will eventually find that vscode is uses `vsce-sign.exe` to verify and installed package.

![@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@](/images/cascade/vsce.png)

We can see that it ran at the same time as the curl command at 11:03:27.164 trying to install `anthropicclaudeassist.claudecode-assist-0.0.1` which is malicious. It's quite common for malicious vs code extension to act as the original and infect computer as in this case.

**Answer: `claudecode-assist`**

**6) What persistence mechanism was established?**

Looking thorugh the logs again using `1948` we can see that it spawned and new cmd process.

![@@@@@@@@@@@@@@@@@@@@@@@@@@](/images/cascade/12488.png)

We can see that this cmd uses reg add to add a new registry key to run `activesyncx86.exe` automatically.

**Answer: `HKCU\SOFTWARE\Microsoft\Windows\CurrentVersion\Run\CitrixActiveSync`**

Thank you and have a nice day

## See also

- [My projects](/projects) — tools I have built
- [Experience](/experience) — my background
