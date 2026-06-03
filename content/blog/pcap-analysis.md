---
title: Pcap analysis - LetsDefend
date: 2025-07-09
tags: ["linux", "networking", "wireshark"]
---

We have captured this traffic from P13’s computer. Can you help him?

The challenge is pretty straight forward you just need basic wireshark knowledge

Let’s take a look at the file

**1.In network communication, what are the IP addresses of the sender and receiver?**

We need to find chat with sender as P13 so we can use a display filter to search for the term `P13`. We can use the contains keyword to check every frame

`> frame contains "P13"`

> Note: When using contains keyword the search term must be enclosed in brackets

We can see many packets lets folow one of the tcp stream, we can see a chat between P13 and Cu713, note the ip addresses

**Ans: 192.168.235.137,192.168.235.131**

**2.P13 uploaded a file to the web server. What is the IP address of the server?**

A web server uses http protocol and the http method will probably be post because http.post method is commonly used for uploading files to a server

So lets put a http.post display filter

`> http.request.method == "POST"`

**Ans: 192.168.1.7**

**3.What is the name of the file that was sent through the network?**

Since we have found the exact packet we can follow it’s tcp stream and search for the line

“Content-Disposition […] filename=’file’ “

**Ans: file**

**4.What is the name of the web server where the file was uploaded?**

If the follow the stream and reach the very bottom we can see that http response where is shows the servername

“Server: Apache/2.4.54 (Win64) OpenSSL/1.1.1p PHP/8.0.25”

**Ans: Apache**

**5.What directory was the file uploaded to?**

In the same response we can see that it says “file uploaded at uploads/file” hence the answer is simply

**Ans: uploads**

**6.How long did it take the sender to send the encrypted file?**

Lets check the stats of the file, we know both the user’s ip address so we can filter it using ip addresses,under the ipv4 tab check until you see the ip address 192.168.1.7 and then check under the duration colum

**Ans: 0.0073**

Thank you and have a nice day

## See also

- [Medium Story](https://medium.com/@pranavsuresh107/pcap-analysis-letsdefend-46aa9382cebb) - same story in medium
- [My projects](/projects) — tools I have built
- [Experience](/experience) — my background
