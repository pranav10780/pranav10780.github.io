---
title: The Report - btlo
date: 2026-09-19
tags: ["soc", "btlo", "writeup"]
---

## Scenario

You are working in a newly established SOC where still there is lot of work to do to make it a fully functional one. As part of gathering intel you were assigned a task to study a threat report released in 2022 and suggest some useful outcomes for your SOC.

Challenge link: [The Report](https://blueteamlabs.online/home/challenge/the-report-a6dd340dba)

```
Download the task file via:
curl -LO https://blueteamlabs.online/storage/files/8c4cbf1af327dca7176473fa355e2dc29cfc527b.zip
```

>NOTE: The password is `BTLO`

## Challenge Submission 

**Question 1) Name the supply chain attack related to Java logging library in the end of 2021 (Format: AttackNickname)**

Before opening any file from the internet (especially pdfs) we need to do a basic check in order to make sure it's safe.

```
remnux@remnux:~/btlo/report/TheReport$ pdfid.py 2022_ThreatDetectionReport_RedCanary.pdf 
PDFiD 0.2.10 2022_ThreatDetectionReport_RedCanary.pdf
 PDF Header: %PDF-1.7
 obj                 1150
 endobj              1150
 stream               256
 endstream            256
 xref                   2
 trailer                2
 startxref              2
 /Page                 80
 /Encrypt               0
 /ObjStm                0
 /JS                    0
 /JavaScript            0
 /AA                    0
 /OpenAction            0
 /AcroForm              0
 /JBIG2Decode           0
 /RichMedia             0
 /Launch                0
 /EmbeddedFile          0
 /XFA                   0
 /Colors > 2^24         0
remnux@remnux:~/btlo/report/TheReport$ sha256sum 2022_ThreatDetectionReport_RedCanary.pdf 
81750245134ce128d2e5f481f8b90fb80267068da7b100b066947624085706bf  2022_ThreatDetectionReport_RedCanary.pdf
remnux@remnux:~/btlo/report/TheReport$ exiftool 2022_ThreatDetectionReport_RedCanary.pdf 
ExifTool Version Number         : 13.50
File Name                       : 2022_ThreatDetectionReport_RedCanary.pdf
Directory                       : .
File Size                       : 11 MB
File Modification Date/Time     : 2022:04:07 10:28:00+00:00
File Access Date/Time           : 2026:09:19 13:18:06+00:00
File Inode Change Date/Time     : 2026:09:19 13:14:48+00:00
File Permissions                : -rw-rw-r--
File Type                       : PDF
File Type Extension             : pdf
MIME Type                       : application/pdf
PDF Version                     : 1.7
Linearized                      : Yes
Language                        : en-US
XMP Toolkit                     : Adobe XMP Core 7.1-c000 79.83fae64, 2022/02/15-08:07:32
Create Date                     : 2022:04:06 10:00:52-06:00
Metadata Date                   : 2022:04:06 10:02:35-06:00
Modify Date                     : 2022:04:06 10:02:35-06:00
Creator Tool                    : Adobe InDesign 17.2 (Macintosh)
Instance ID                     : uuid:e99a23ff-3b50-7348-bce7-9ea333bb5eb0
Original Document ID            : xmp.did:249dee16-d64a-4d45-954c-b7fc412f0218
Document ID                     : xmp.id:53f1ac26-6365-49c9-8fd8-9e0b153d326a
Rendition Class                 : proof:pdf
Derived From Instance ID        : xmp.iid:abfbc52d-67a2-47d1-8303-972e37b0c698
Derived From Document ID        : xmp.did:2de05f11-26c5-4bca-a95d-147d0f3396ea
Derived From Original Document ID: xmp.did:249dee16-d64a-4d45-954c-b7fc412f0218
Derived From Rendition Class    : default
History Action                  : converted
History Parameters              : from application/x-indesign to application/pdf
History Software Agent          : Adobe InDesign 17.2 (Macintosh)
History Changed                 : /
History When                    : 2022:04:06 10:00:53-06:00
Format                          : application/pdf
Producer                        : Adobe PDF Library 16.0.7
Trapped                         : False
Page Count                      : 80
Creator                         : Adobe InDesign 17.2 (Macintosh)
remnux@remnux:~/btlo/report/TheReport$ file 2022_ThreatDetectionReport_RedCanary.pdf 
2022_ThreatDetectionReport_RedCanary.pdf: PDF document, version 1.7 (zip deflate encoded)
```

We can see nothing suspicious in the metadata lets give the hash to virus total to be safe, just in case.

![virustotal](/images/the_report/virustotal.png)

As you can see this file is safe to open.

After reading through the file we will eventually stumble upon the section `Supply chain compromise` under the `Trends` section.

![log4j](/images/the_report/log4j.png)

**Answer: `Log4j`**

**Question 2) Mention the MITRE Technique ID which effected more than 50% of the customers (Format: TXXXX)**

We can sit there and read the entire report or press `ctrl + f` to search for keyword `TOP TECHNIQUES`.

![mitre](/images/the_report/mitre.png)

**Answer: `T1059`**

**Question 3) Submit the names of 2 vulnerabilities belonging to Exchange Servers (Format: VulnNickname, VulnNickname)**

Using the `ctrl + f` and using the keywords `Exchange Server` we wil get the result.

![server](/images/the_report/server.png)

**Answer: `proxylogon, proxyshell`**

**Question 4) Submit the CVE of the zero day vulnerability of a driver which led to RCE and gain SYSTEM privileges (Format: CVE-XXXX-XXXXX)**

Since the format of a cve is always going to be starting with `CVE-` lets search for it.

![cve](/images/the_report/cve.png)

**Answer: `CVE-2021-34527`**

**Mention the 2 adversary groups that leverage SEO to gain initial access (Format: Group1, Group2)**

Searching for keywords `SEO`.

![seo](/images/the_report/seo.png)

**Answer: `Gootkit, Yellow Cockatoo`**

**Question 6) In the detection rule, what should be mentioned as parent process if we are looking for execution of malicious js files [Hint: Not CMD] (Format: ParentProcessName.exe)**

Using the keyword `parent`.

![wcript](/images/the_report/wscript.png)
![hcrypt](/images/the_report/hcrypt.png)

Hcrypt is mainly lauched via javascript and uses `wscript.exe` as parent process.

**Answer: `wscript.exe`**

**Question 7) Ransomware gangs started using affiliate model to gain initial access. Name the precursors used by affiliates of Conti ransomware group (Format: Affiliate1, Affiliate2, Afilliate3)**

We can search for `ransomware group` and get a table with our answers.

![conti](/images/the_report/conti.png)

**Answer: `Qbot,Bazar,IcedID`**

**Question 8) The main target of coin miners was outdated software. Mention the 2 outdated software mentioned in the report (Format: Software1, Software2)**

Search for keyword `outdated`.

![outdated](/images/the_report/outdated.png)

**Answer: `Jboss, weblogic`**

**Question 9) Name the ransomware group which threatened to conduct DDoS if they didn't pay ransom (Format: GroupName)**

Search for keyword `ddos`.

![fancy](/images/the_report/fancy.png)

**Answer: `Fancy Lazarous`**

**Question 10) What is the security measure we need to enable for RDP connections in order to safeguard from ransomware attacks? (Format: XXX)**

Search for keyword `rdp`.

![mfa](/images/the_report/mfa.png)

**Answer: `mfa`**

Thank you and have a nice day

## See also

- [My projects](/projects) — tools I have built
- [Experience](/experience) — my background
