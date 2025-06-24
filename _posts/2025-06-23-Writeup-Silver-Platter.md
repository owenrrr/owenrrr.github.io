---
layout: post
title:  "TryHackMe Challenge Writeup: Silver Platter"
date:   23 June 2025
categories: Demo
tags: TryHackMe-Challenge Writeup
---
<html>
<body>
<div markdown="block" style="margin-top: 10px">
    
### Intro
本篇是關於TryHackMe上的Challenge: [Silver Platter](https://tryhackme.com/room/silverplatter) 我的解題思路和歷程，紀錄自己嘗試了哪些手段以及學習了哪些新的攻擊手法和切入角度
  
### Step 1. Nmap
- Nmap先觀察開了哪些端口和服務：發現了22/80/8080
- 首先，22肯定不是切入點，所以我先查看80，再來8080

### Step 2. 觀察port80
- 80指向一個網站，進去快速瀏覽後發現有一組可能有用的用戶名和關聯的平台名
- 記下: `scr1ptkiddy` 和 `Silverpeas`

### Step3. 觀察port8080
- 8080也指向一個網站，只不過直接輸入根目錄會得到404
- 嘗試使用gobuster搭配 /seclists/Discovery/Web-Content/directory-xx.txt 去爆破可用目錄，沒找到什麼有用的東西
- 記得之前在80的時候記的一組用戶名和平台名字，嘗試當作目錄(這邊卡超久，一直在嘗試/Silverpeas，結果最後發現要小寫 /silverpeas)
- 順利找到8080

### Step 4. 嘗試登入
- 目前知道了用戶名為 `scr1ptkiddy` 但完全不知道密碼
- 在題目中也說明了使用常見的 rockyou.txt 沒辦法爆破密碼
- 那麼嘗試更換思路，查詢Silverpeas相關漏洞，就直接發現了 [CVE-2024-36042](https://www.cve.org/CVERecord?id=CVE-2024-36042)
- 簡單來說就是在登入時去除掉Password參數就能直接登入
- 那麼就用Burp-suite抓登入時的請求，修改並Forward
- 順利登入

### Step 5. 觀察Silverpeas網站
- 簡單瀏覽後發現有一則通知，到自己的設置下觀察URL形式發現在詳細查看一則通知時URL形式用ID=x來當查詢條件
- 因此同樣使用Burp-suite修改ID值，發現ID=6時會看到tim登入的帳密
- 成功獲取User權限

### Step 6. 提權
- 使用tim身分ssh登入
- id查看tim的權限有哪些，發現tim位於一個adm群組中，猜測是administrator
- 查看這個群組有哪些權限(針對讀/r): `find / -group adm -perm -g=r 2>/dev/null`
- 發現可以查看 `/usr/log` 和 `/etc/passwd` 以及其他的位置，反正也沒什麼其他思路就都看看

### Step 7. 到處看看
- 首先觀察 `/etc/passwd`：雖然現在密碼都儲存在 `/etc/shadow`，但是 `/etc/passwd` 還是會存用來儲存所有系統使用者帳號的資訊，我就觀察到有一個用戶 `tyler` 的使用者全名是root，可能可以作為提權切入點:
  
```bash
tyler:x:1000:1000:root:/home/tyler:/bin/bash
```  
  
- tyler: 用戶名
- x: 表示密碼存放位置,x表示/etc/shadow
- 1000: uid
- 1000: guid
- root: 使用者全名/備註
- /home/tyler: 主目錄
- /bin/bash: 登錄後默認的shell
  
- 再來觀察 `/usr/log` 下有沒有感興趣的帳密信息
- 用 `cd /usr/log && grep -iR password`
- 成功看到用戶tyler在POSTGRESQL的帳密，嘗試登入
- 成功獲取Root權限


## 學習資料
1. [TryHackMe Challenge: Silver Platter](https://tryhackme.com/room/silverplatter) 
2. [Silver Platter Writeup](https://0xb0b.gitbook.io/writeups/tryhackme/2025/silver-platter)

</div>
</body>
</html>
