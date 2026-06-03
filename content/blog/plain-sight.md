---
title: I3a4dam’s Plain sight - crackmes.one
date: 2025-09-29
tags: ["linux", "ctf", "hacking"]
---

This is a pretty easy crack me to do today

We first need to download the `6513567328b5870bef263329.zip` file from the website called [crackmes.one](https://crackmes.one/crackme/6513567328b5870bef263329)

![download](/images/plainsight/1.webp)

> NOTE: No need to use wget to download the file, it is optional

We need to unzip this file

![unzip](/images/plainsight/2.webp)

**NOTE: Password: crackmes.one**

Unzipping gives us a new file named `plain_sight.zip`, we need to unzip this file too

![unzip](/images/plainsight/3.webp)

> NOTE: Password: crackmes.one

> NOTE: You need to have the p7zip-full package installed

This will give us a binary executable

![exe](/images/plainsight/4.webp)

We can see the file details here, lets statically analyze the file using strings

The string command basically extracts all the data present inside a file which is in ascii format and shows it as a list, it is useful for static analysis

`> strings plain_sight | less`

![string](/images/plainsight/5.webp)

We can see a string towards the bottom of the list called **do_not_hardcode**, this is the answer

This also serves as a powerful remind to never hardcode string into a binary file

Thank you and have a nice day

## See also

- [Medium Story](https://medium.com/@pranavsuresh107/i3a4dams-plain-sight-crackmes-one-403e6cd4749a) - same story in medium
- [My projects](/projects) — tools I have built
- [Experience](/experience) — my background
