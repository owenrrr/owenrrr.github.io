---
layout: post
title:  "TryHackMe Challenge Writeup: Creative"
date:   1 July 2025
categories: Demo
tags: TryHackMe-Challenge Writeup
---
<html>
<body>
<div markdown="block" style="margin-top: 10px">
    
### Intro
本篇是關於TryHackMe上的Challenge: [Creative](https://tryhackme.com/room/creative) 我的解題思路和歷程，紀錄自己嘗試了哪些手段以及學習了哪些新的攻擊手法和切入角度

### Step 1. Nmap

```bash
PORT   STATE SERVICE
22/tcp open  ssh
| ssh-hostkey: 
|   3072 1a:a5:be:be:61:96:d0:bd:d8:8a:54:f9:92:e0:5b:fb (RSA)
|   256 dd:f6:c3:ce:41:03:65:d8:1d:0e:3e:bb:58:9d:c3:7e (ECDSA)
|_  256 fc:bc:54:30:8d:67:35:23:6f:fc:2d:a0:78:cc:5a:85 (ED25519)
80/tcp open  http
| http-methods: 
|_  Supported Methods: GET HEAD POST OPTIONS
|_http-title: Did not follow redirect to http://creative.thm
```

```bash
└─# nmap -p80 -sVC -O creative.thm
Starting Nmap 7.95 ( https://nmap.org ) at 2025-06-30 04:38 EDT
Nmap scan report for creative.thm (10.10.253.240)
Host is up (0.48s latency).

PORT   STATE SERVICE VERSION
80/tcp open  http    nginx 1.18.0 (Ubuntu)
|_http-server-header: nginx/1.18.0 (Ubuntu)
|_http-title: Creative Studio | Free Bootstrap 4.3.x template
Warning: OSScan results may be unreliable because we could not find at least 1 open and 1 closed port
Device type: general purpose
Running (JUST GUESSING): Linux 4.X|2.6.X|3.X|5.X (97%)
OS CPE: cpe:/o:linux:linux_kernel:4.15 cpe:/o:linux:linux_kernel:2.6 cpe:/o:linux:linux_kernel:3 cpe:/o:linux:linux_kernel:5
Aggressive OS guesses: Linux 4.15 (97%), Linux 2.6.32 - 3.13 (91%), Linux 3.10 - 4.11 (91%), Linux 3.2 - 4.14 (91%), Linux 4.15 - 5.19 (91%), Linux 5.0 - 5.14 (91%), Linux 2.6.32 - 3.10 (91%), Linux 5.4 (90%)
```

- 檢查80

### Step 2. Web
- 發現幾位開發者名字可能有用，先記起來
- 用 whatweb 跑跑看:

```bash
└─# whatweb creative.thm
http://creative.thm [200 OK] Bootstrap, Country[RESERVED][ZZ], Email[info@example.com,info@website.com], Frame, HTML5, HTTPServer[Ubuntu Linux][nginx/1.18.0 (Ubuntu)], IP[10.10.253.240], JQuery[3.4.1], Meta-Author[Devcrud], PasswordField, Script, Title[Creative Studio | Free Bootstrap 4.3.x template], YouTube, nginx[1.18.0]
```

- 發現網站作者名爲 **Devcrud**，記下來或許後面用得到
- 嘗試路徑枚舉，沒發現什麽感興趣的:

```bash
└─# gobuster dir -u creative.thm -w /usr/share/seclists/Discovery/Web-Content/directory-list-2.3-medium.txt -t 50 -x html
```

- 嘗試 subdomain 枚舉，因爲發現其實原本的網站域名是 creative.thm 還可以有子域名，發現一堆結果但是有一個返回的頁面跟別的不同: `beta.creative.thm`

```bash
└─# ffuf -u http://10.10.255.41 -H "Host: FUZZ.creative.thm" -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-20000.txt -r -c
beta                    [Status: 200, Size: 591, Words: 91, Lines: 20, Duration: 502ms]
mobile                  [Status: 200, Size: 37589, Words: 14867, Lines: 686, Duration: 511ms]
admin                   [Status: 200, Size: 37589, Words: 14867, Lines: 686, Duration: 519ms]
mail2                   [Status: 200, Size: 37589, Words: 14867, Lines: 686, Duration: 516ms]
old                     [Status: 200, Size: 37589, Words: 14867, Lines: 686, Duration: 521ms]
new                     [Status: 200, Size: 37589, Words: 14867, Lines: 686, Duration: 522ms]
cp                      [Status: 200, Size: 37589, Words: 14867, Lines: 686, Duration: 524ms]
support                 [Status: 200, Size: 37589, Words: 14867, Lines: 686, Duration: 528ms]
cpanel                  [Status: 200, Size: 37589, Words: 14867, Lines: 686, Duration: 529ms]
shop                    [Status: 200, Size: 37589, Words: 14867, Lines: 686, Duration: 529ms]
...
```

### Step 3. Web(beta)



### 自我總結


## 學習資料
1. [TryHackMe Challenge: Opacity](https://tryhackme.com/room/opacity)
2. [Writeup 1: Opacity](https://github.com/0xDeco/CTF-Notes/tree/main/Opacity)
3. [Writeup 2: Opacity](https://www.youtube.com/watch?v=oTWxXR7QywY)

</div>
</body>
</html>
