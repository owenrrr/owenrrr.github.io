---
layout: post
title:  "Hack The Box: Planning Writeup"
date:   17 Sep 2025
categories: Demo
tags: Writeup
---
<html>
<body>
<div markdown="block" style="margin-top: 10px">
    
### Intro
You can find out the challenge on [Hack The Box: Planning](https://app.hackthebox.com/machines/Planning)



### Walkthrough

- First, I use nmap to scan opened common ports on this machine

```bash
└─$ nmap -sT -sV -p- -T4 10.10.11.68                        
Starting Nmap 7.95 ( https://nmap.org ) at 2025-09-17 15:56 MST
Stats: 0:00:11 elapsed; 0 hosts completed (1 up), 1 undergoing Connect Scan
Connect Scan Timing: About 80.73% done; ETC: 15:56 (0:00:02 remaining)
Nmap scan report for 10.10.11.68
Host is up (0.099s latency).
Not shown: 998 closed tcp ports (conn-refused)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 9.6p1 Ubuntu 3ubuntu13.11 (Ubuntu Linux; protocol 2.0)
80/tcp open  http    nginx 1.24.0 (Ubuntu)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 21.40 seconds
```

- add a domain mapping entry on `/etc/hosts`, then access `http://planning.htb/` on the browser

- use `whatweb` to check any framework versions that the website is using; Moreover, I see .php file on the website so I try `wpscan` and apparently is not using WordPress

```bash
└─# whatweb http://planning.htb                                          
http://planning.htb [200 OK] Bootstrap, Country[RESERVED][ZZ], Email[info@planning.htb], HTML5, HTTPServer[Ubuntu Linux][nginx/1.24.0 (Ubuntu)], IP[10.10.11.68], JQuery[3.4.1], Script, Title[Edukate - Online Education Website], nginx[1.24.0]
```

- try `gobuster` or `ffuf` to check subdomain under the domain `planning.htb`. Found the special subdomain **grafana.planning.htb**

```bash
└─# ffuf -u "http://10.10.11.68" -H "Host: FUZZ.planning.htb" -w /usr/share/seclists/Discovery/DNS/namelist.txt -t 100 -ac 

        /'___\  /'___\           /'___\       
       /\ \__/ /\ \__/  __  __  /\ \__/       
       \ \ ,__\\ \ ,__\/\ \/\ \ \ \ ,__\      
        \ \ \_/ \ \ \_/\ \ \_\ \ \ \ \_/      
         \ \_\   \ \_\  \ \____/  \ \_\       
          \/_/    \/_/   \/___/    \/_/       

       v2.1.0-dev
________________________________________________

 :: Method           : GET
 :: URL              : http://10.10.11.68
 :: Wordlist         : FUZZ: /usr/share/seclists/Discovery/DNS/namelist.txt
 :: Header           : Host: FUZZ.planning.htb
 :: Follow redirects : false
 :: Calibration      : true
 :: Timeout          : 10
 :: Threads          : 100
 :: Matcher          : Response status: 200-299,301,302,307,401,403,405,500
________________________________________________

grafana                 [Status: 302, Size: 29, Words: 2, Lines: 3, Duration: 74ms]
:: Progress: [151265/151265] :: Job [1/1] :: 1308 req/sec :: Duration: [0:01:54] :: Errors: 0 ::
```

- Access `grafana.planning.htb` and there's a login page. Check the website carefully and found that the website is using a framework called Grafana with v11.0.0. Furthermore, I searched Grafana v11.0.0 vulnerability and found there's a CVE-2024-9264 which can cause arbitrary file read, command execution on either shell commands or DuckDB commands.

- Found the exploit code from [https://github.com/nollium/CVE-2024-9264](https://github.com/nollium/CVE-2024-9264). Let's try out!




## 學習資料
1. [TryHackMe Challenge: Creative](https://tryhackme.com/room/creative)
2. [Writeup \| Creative](https://0xb0b.gitbook.io/writeups/tryhackme/2024/creative)
3. [使用LD_Preload的Linux权限升级技巧](https://www.cnblogs.com/backlion/p/10503985.html)

</div>
</body>
</html>
