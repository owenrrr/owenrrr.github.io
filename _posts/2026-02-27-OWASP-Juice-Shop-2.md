---
layout: post
title:  "OWASP Juice Shop Walkthrough (2)"
date:   2025-02-27 11:15:55 +0800
categories: Demo
tags: OWASP-Juice-Shop
---
<html>
<body>
<div markdown="block" style="margin-top: 10px">
    
### Introduction:
In this writeup, we will start from the beginning of pwning OWASP Juice Shop to a comprehensive walkthrough. Let's don't talk too much and get started! Here's the OWASP Juice Shop official document: [OWASP Juice Shop](https://pwning.owasp-juice.shop/companion-guide/latest/index.html).
This article includes all two star challenges!

<br><br>

### Outline

| Challenge | Vulnerability |
| --- | --- |
| [Exposed Credentials](#21-exposed-credentials) | Sensitive Data Exposure |
| [Login Admin](#22-login-admin) | SQL Injection |
| [Admin Section](#23-admin-section) | Broken Access Control | 
| [Password Strenght](#24-password-strendth) | Broken Authentication | 
| [View Basket](#25-view-basket) | Broken Access Control | 
| [Deprecated Interface](#26-deprecated-interface) | Security Misconfiguration |  
| [Empty User Registration](#27-empty-user-registration) | Improper Input Validation |  
| [Five Star Feedback](#28-five-star-feedback) | Broken Access Control |  
| [Login MC SafeSearch](#29-login-mc-safesearch) | Sensitive Data Exposure |  
| [Meta Geo Stalking](#210-meta-geo-stalking) | Sensitive Data Exposure |  
| [NFT Takeover](#211-nft-takeover) | Sensitive Data Exposure |  
| [Security Policy](#212-security-policy) | Misc. |  
| [Visual Geo Stalking](#213-visual-geo-stalking) | Sensitive Data Exposure | 
| [Weird Crypto](#214-weird-crypto) | Cryptographic Issues | 

<br><br>

### 2 Star Challenge
#### 2.1 Exposed Credentials
```
A developer was careless with hardcoding unused, but still valid credentials for a testing account on the client-side.
```

While we did one-star challenges viewing `main.js`, we found something interesting that there's a testing account revealed in multiple places. `testingUsername="testing@juice-sh.op"; testingPassword="IamUsedForTesting"`.

<br>

#### 2.2 Login Admin
```
Log in with the administrator's user account.
```

When we browse through the About me page, there's a customer comment that catches my eye with the email `***in@juice-sh.op`. Combined with the admin's account, we can take a easy guess that admin's email address is `admin@juice-sh.op`. As long as we don't know the password is, we can use SQL injection to bypass the login authentication.

<br>

#### 2.3 Admin Section
```
Access the administration section of the store.
```

Try `/administration`, `admin` and `administrator`. The `/administration` is valid.

<br>

#### 2.4 Password Strendth
```
Log in with the administrator's user credentials without previously changing them or applying SQL Injection.
```

Of course we intercepts login requests using burp suite. We note that this request uses `/rest/user/login` path and `Content-Type: application/json` form data (which is different from the default data type used by hydra: application/x-www-form-urlencoded). Therefore, we can use hydra to enumerate password. Here's the command to enumerate admin's password. We've tried out several wordlists(xato-100~xato-100000) and successfully got the password.

```bash
hydra -l admin@juice-sh.op -P /usr/share/seclists/Passwords/xato-net-10-million-passwords-100000.txt -s 3000 localhost http-post-form '/rest/user/login:{"email"\:"^USER^","password"\:"^PASS^"}:H=Content-Type\:application/json:1=:S=200' -V -t 20
```

<br>

#### 2.5 View Basket
```
View another user's shopping basket.
```

Again, we use burp suite to intercept the request of viewing basket. The following is what the request looks like:
```
GET /rest/basket/6 HTTP/1.1
Host: localhost:3000
User-Agent: Mozilla/5.0 (X11; Linux x86_64; rv:128.0) Gecko/20100101 Firefox/128.0
Accept: application/json, text/plain, */*
Accept-Language: en-US,en;q=0.5
Accept-Encoding: gzip, deflate, br
...
```

The number in the URL looks like the userid which controls whose basket should be displayed. We then tried to modify the number: `/rest/basket/7` and forward it to the website. As what we thought, it successfully displays another user's basket.

<br>

#### 2.6 Deprecated Interface
```
Use a deprecated B2B interface that was not properly shut down.
```

This one is more difficult to me. As usual, we search for any deprecated functions or protals in JS files because most of developers won't immediately remove all deprecated functions clearly. Therefore, we can try searching "B2B" on all JS files and find that there's only one result relating to our goal.
```
Input area for uploading a single invoice PDF or XML B2B order file or a ZIP archive containing multiple invoices or orders.
```

There's one place that supports users to upload their invoices that is the Customer Feedback page. As the result we found combined with the description of the challenge, we shouldn't be able to upload B2B file in XML format(deprecated). Hence, we need to figure out how to bypass it. We observed the way that the website used to block deprecated B2B file relies on the front-end attribute "accept='.pdf,.zip'". As a result, we can simply modify the DOM attribute to "accept='.xml'" to bypass this mechanism.

Furthermore, we can still use Burp Suite to first upload a file in pdf format, intercept it, and modify the file format to XML.

<br>

#### 2.7 Empty User Registration
```
Register a user with an empty email and password.
```

Again, we use burp suite to intercept the request of viewing basket. The following is what the request looks like. We can simply modify the JSON data with empty email and password, finally forwards it to the website and complete the challenge.
```
POST /api/Users/ HTTP/1.1
Host: localhost:3000
...
{"email":"demo@gmail.com","password":"demo1","passwordRepeat":"demo1","securityQuestion":{"id":1,"question":"Your eldest siblings middle name?","createdAt":"2026-02-27T16:07:44.898Z","updatedAt":"2026-02-27T16:07:44.898Z"},"securityAnswer":"1"}
```

<br>

#### 2.8 Five-Star Feedback
```
Get rid of all 5-star customer feedback.
```

This one is easy. Just simply access admin portal(`/administration`) and remove a 5-star customer feedback.

<br>

#### 2.9 Login MC SafeSearch
```
Log in with MC SafeSearch's original user credentials without applying SQL Injection or any other bypass.
```

As we access admin's portal(`administration`), we note a MC SafeSearch account whose email is `mc.safesearch@juice-sh.op`. MC SafeSearch is a rapper who produced a song called "Protect Ya' Passwordz". In the song, we got a clue what his password would be. He said he would use his favorite pet's name as password, and he doesn't bother if you know this pattern because he will change all letter "o" into number "0". As what he said, his pet's name is "Mr. Noodles".

<br>

#### 2.10 Meta Geo Stalking
```
Determine the answer to John's security question by looking at an upload of him to the Photo Wall and use it to reset his password via the Forgot Password mechanism.
```

As we viewed the Photo Wall page, we saw an image from "j0hNny" with "I love going hiking here...". Since we don't have any additional information on website source code(DOM), we tried downloading the image and use `exiftool` to extract the metadata of this image. We got the result as following:
```
└─# exiftool Downloads/favorite-hiking-place.png 
ExifTool Version Number         : 13.10
File Name                       : favorite-hiking-place.png
Directory                       : Downloads
File Size                       : 667 kB
File Modification Date/Time     : 2026:02:27 11:34:58-07:00
File Access Date/Time           : 2026:02:27 11:37:40-07:00
File Inode Change Date/Time     : 2026:02:27 11:34:58-07:00
File Permissions                : -rw-rw-r--
File Type                       : PNG
File Type Extension             : png
MIME Type                       : image/png
Image Width                     : 471
Image Height                    : 627
Bit Depth                       : 8
Color Type                      : RGB
Compression                     : Deflate/Inflate
Filter                          : Adaptive
Interlace                       : Noninterlaced
Exif Byte Order                 : Little-endian (Intel, II)
Resolution Unit                 : inches
Y Cb Cr Positioning             : Centered
GPS Version ID                  : 2.2.0.0
GPS Latitude Ref                : North
GPS Longitude Ref               : West
GPS Map Datum                   : WGS-84
Thumbnail Offset                : 224
Thumbnail Length                : 4531
SRGB Rendering                  : Perceptual
Gamma                           : 2.2
Pixels Per Unit X               : 3779
Pixels Per Unit Y               : 3779
Pixel Units                     : meters
Image Size                      : 471x627
Megapixels                      : 0.295
Thumbnail Image                 : (Binary data 4531 bytes, use -b option to extract)
GPS Latitude                    : 36 deg 57' 31.38" N
GPS Longitude                   : 84 deg 20' 53.58" W
GPS Position                    : 36 deg 57' 31.38" N, 84 deg 20' 53.58" W
```

We got the GPS Position which is **36 deg 57' 31.38" N, 84 deg 20' 53.58" W**. We then search on Google for where it indicates - **Daniel Boone National Forest**. Therefore, the answer for his security question "What's your favorite place to go hiking" is revealed.

<br>

#### 2.11 NFT Takeover
```
Take over the wallet containing our official Soul Bound Token (NFT).
```

For this challenge, we should know some basic knowledge for NFT. The entrypoint of this challenge is on customer's feedback where `Please send me the juicy chatbot NFT in my wallet at /juicy-nft : "purpose betray marriage blame crunch monitor spin slide donate sport lift clutch" (***ereum@juice-sh.op)`. The portal is on `/juicy-nft` and the secret recovery phrase is the 12-words string. We can then use [Seed Phrase to Private Key Tool](https://www.iancoleman.net/ethereum-bip39/) to generate the private key for the seed.

Here're some useful articles about NFT and its related concepts and terminologies that you could refer:
- [What are NFTs?](https://ethereum.org/nft/)
- [Ethereum accounts](https://ethereum.org/developers/docs/accounts/)
- [Keys in proof-of-stake Ethereum](https://ethereum.org/developers/docs/consensus-mechanisms/pos/keys/)

<br>

#### 2.12 Security Policy
```
Behave like any "white-hat" should before getting into the action.
```

As described in [Wikipedia](https://en.wikipedia.org/wiki/Security.txt), *security.txt is an accepted standard for website security information that allows security researchers to report security vulnerabilities easily*. The path of `security.txt` is defined in the industry standard [RFC8615](https://datatracker.ietf.org/doc/html/rfc8615) which should be placed under `/.well-known/` directory. The following is what we got on `/.well-known/security.txt`:

```
Contact: mailto:donotreply@owasp-juice.shop
Encryption: https://keybase.io/bkimminich/pgp_keys.asc?fingerprint=19c01cb7157e4645e9e2c863062a85a8cbfbdcda
Acknowledgements: /#/score-board
Preferred-languages: en, ar, az, bg, bn, ca, cs, da, de, ga, el, es, et, fi, fr, ka, he, hi, hu, id, it, ja, ko, lv, my, nl, no, pl, pt, ro, ru, si, sv, th, tr, uk, zh
Hiring: /#/jobs
Csaf: http://localhost:3000/.well-known/csaf/provider-metadata.json
Expires: Sat, 27 Feb 2027 16:08:00 GMT
```

<br>

#### 2.13 Visual Geo Stalking
```
Determine the answer to Emma's security question by looking at an upload of her to the Photo Wall and use it to reset her password via the Forgot Password mechanism
```

The security question for Emma is "Company you first work for as an adult". Let's give it a try on search engine with directly using this image. We got a location **Kenaupark, Haarlem**. After that, we examined carefully on Google Map and got the exact address of the building in the image where is **Kenaupark 23**. However, we get stuck on this. We kept searching the company located in this building and got two results: "Blank & Partners B.V." and "HCC". I tried many times with different combinations, however, it failed.

After that, I was wondering why this challenge named "Visual Geo Stalking"? Do we have to look carefully on this image somehow? So we took a closer look at this picture and finally got some ideas. **ITsec** appears in a vague way.

<br>

#### 2.14 Weird Crypto
```
Inform the shop about an algorithm or library it should definitely not use the way it does.
```

In **Meta Geo Stalking** challenge, after we input the coorect answer for John's security question, we got a HTTP response including a suspicious password hash. 
```
HTTP/1.1 200 OK
...
{"user":{"id":18,"username":"j0hNny","email":"john@juice-sh.op","password":"827ccb0eea8a706c4c34a16891f84e7b","role":"customer","deluxeToken":"","lastLoginIp":"","profileImage":"assets/public/images/uploads/default.svg","totpSecret":"","isActive":true,"createdAt":"2026-02-27T16:07:44.908Z","updatedAt":"2026-02-27T18:51:51.447Z","deletedAt":null}}
```

We can easily tell that it is insecure because of its length and therefore use some tools(`hashid`,`hashcat`) to identify which encryption algorithm it's using. After that, we then use cracking tools([Crackstation](https://crackstation.net/)) to get the plaintext.
```bash
└─# hashid 827ccb0eea8a706c4c34a16891f84e7b
Analyzing '827ccb0eea8a706c4c34a16891f84e7b'
[+] MD2 
[+] MD5 
[+] MD4 

└─# hashcat --identify "827ccb0eea8a706c4c34a16891f84e7b"
The following 11 hash-modes match the structure of your input hash:

      # | Name                                                       | Category
  ======+============================================================+======================================
    900 | MD4                                                        | Raw Hash
      0 | MD5                                                        | Raw Hash
     70 | md5(utf16le($pass))                                        | Raw Hash
```

Anyway, back to the main topic, as the description said, we need to inform the shop by submitting a customer feedback. Here we type **md5** in the textbox.

<br><br>

## Materials
1. [OWASP Juice Shop Official Document](https://pwning.owasp-juice.shop/companion-guide/latest/index.html)
2. [What are NFTs?](https://ethereum.org/nft/)
3. [Ethereum accounts](https://ethereum.org/developers/docs/accounts/)
4. [Keys in proof-of-stake Ethereum](https://ethereum.org/developers/docs/consensus-mechanisms/pos/keys/)

</div>
</body>
</html>
