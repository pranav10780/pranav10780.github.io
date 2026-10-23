---
title: Encrypted email
description: "How to send a encrypted email"
date: 2026-10-23
tags: ["email", "encryption", "gpg"]
---

In this blog i am going to show you how to send an encrypted email to ME!.

![contact](/images/encryption/contact.png)

In the above image we can see that my public keys are available at keys.openpgp.org.

## Importing keys

Inorder to use my public keys to encrypt a message we need to import it to the gpg keyring

We can either use the keys.openpgp.org to fetch the keys or directly download it from my website.

```
### keys.openpgp.org
gpg --keyserver hkps://keys.openpgp.org --recv-keys 9306FBEA4A7C3A78C5EAA4005B6264FC51180427

### from my website
curl -s https://pranav10780.github.io/pranav.asc | gpg --import
```

## Verify keys

This step is not necessary but just in case you can use it to verify the keys

```
gpg --fingerprint 9306FBEA4A7C3A78C5EAA4005B6264FC51180427
```

## Encryption

In order to encrypt something we need a message

```
pranav@server:$ echo "hello world" > message.txt
pranav@server:$ cat message.txt 
hello world
```

This is going to be our secret message, now we can encrypt it

```
pranav@server:$gpg --armor --encrypt --recipient 9306FBEA4A7C3A78C5EAA4005B6264FC51180427 message.txt
pranav@server:$ cat message.txt.asc 
-----BEGIN PGP MESSAGE-----

hF4D/9dX6sAVRWASAQdAVmTOtblUHUQcqUVsCAxo2eiDUlnw+5rB6QkQ/FE/VXIw
YV322kpzd3O6YD4ZGWLmgkapVlL60lABHtEbbwsMUaNB57BlyrVZJ3BfeVPpXeC1
1FwBCQIQRU6/xH3iETvYUo/JhQDtjvFEDSxaPbAXMqyO1VEFQCFQBQJebdZTqxYP
Ao+dpFwLK3nPLfXQsikK3W0GBEZlMPWskEApMBbOPkTfX98mQ5fyxKqiRfIjfw==
=v1rw
-----END PGP MESSAGE-----
```

This is our encrypted message which is safe to be transmitted by email services like gmail without being spied by google.

## Decryption

>NOTE: If the message is encrypted by my key then i can only decrypt it

```
pranav@server:$ gpg --decrypt message.txt.asc
gpg: encrypted with cv25519 key, ID FFD757EAC0154560, created 2026-10-23
      "Pranav Suresh <pranavsuresh107@gmail.com>"
hello world
```

As you can see i was able to decrypt the encrypted message!

Hope you learned something

Thank you and have a nice day

## See also

- [My projects](/projects) — tools I have built
- [Experience](/experience) — my background
