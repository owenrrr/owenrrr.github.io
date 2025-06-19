---
layout: post
title:  "John the Ripper"
date:   19 June 2025
categories: Demo
tags: Cybersecurity, Password-Cracking-Tool
---
<html>
<body>
<div markdown="block" style="margin-top: 10px">
    
### Intro
***John the ripper*** 相信大家可能或多或少都有聽過這個密碼/哈希值破解工具，跟 ***Hashcat*** 一起使用應該就可以應付大多數的破解場景了，因此本篇先介紹 *John the ripper*，後續再來介紹 *Hashcat* 。

這兩款工具都支援非常多類型的哈希算法，下面就直接介紹怎麽使用以及我自己常用的一些指令(*然後我發現附工具的完整幫助界面有點太冗餘所以直接以介紹常用的爲主，主打的就是實用 :D* )：
  
  
### 經常使用
```bash
# 如果不確定是使用的是什麽算法
hashid -j 5f4dcc3b5aa765d61d8327deb882cf99
# 找有沒有支援特定算法
john --list=formats | grep sha512
# 找--list可以看哪些參數支援哪些值
john --list=help
# 基本使用
john --format=raw-md5 --wordlist=rockyou.txt hash.txt
# 讓john自動判斷格式並破解
john hash.txt
# 顯示破解完后的資訊
john --show hash.txt
# 使用single模式(根據用戶名生成相關密碼字典檔破解)
john --single --format=raw-md5 hash.txt
# 使用規則增强破解效果
john --rules hash.txt
# 從上一次中斷處繼續
john --restore
# 破解需要密碼的zip
zip2john secure.zip > zip_hash
john --wordlist=rockyou.txt zip_hash
# 破解需要密碼的rar
rar2john secure.rar > rar_hash
john --wordlist=rockyou.txt rar_hash
# 破解需要密碼的id_rsa(passphrase)
ssh2john id_rsa > id_rsa_hash
john --wordlist=rockyou.txt id_rsa_hash
```
  
- `zip2john`, `rar2john`, `ssh2john` 都是 John the ripper jumbo version才有支持的功能，所以若是沒有找到這些指令的話可以先檢查版本
- `--single`模式算是 *john* 很特別的一個特色，它可以根據輸入的用戶名來猜測並生成相應的字典檔來破解，完全符合社交工程和真實世界中用戶密碼的生成規則，而并非單純暴力破解或不相干的字典檔。但要注意在這個模式下，輸入的檔案需要包含用戶名，即若是md5這種原本就不包含用戶名的hash算法就需要修改一下：
  - 例如我知道這個 *MD5* 值是屬於 *Lucy* 的
  - 則原本hash.txt中包含 `5f4dcc3b5aa765d61d8327deb882cf99`
  - 就需要改寫為 `lucy:5f4dcc3b5aa765d61d8327deb882cf99`
  - 另外要注意，`--single`模式下要指定 `--format`格式，因爲它不會自動判別；如果沒有指定會報錯
- `--rules` 或 `--rule=xxx` 指定應用特定規則，通常可在 `/etc/john/john.conf` 中查詢所有規則，還可以有自定義規則供後續使用，詳情可參考 [這裏](https://www.openwall.com/john/doc/RULES.shtml)
  
  
### 小結
總而言之，`john` 算是一款非常實用的工具，它可以被用在滲透測試中:
- **/etc/shadow**
- **Windows SAM**
- **zip/rar passphrase**
- **id_rsa passphrase**
- **hash via SQLi/LFI**
- **Kerberos TGT/TGS**
- **社交工程密碼猜測**  
  
所以如果還沒學會這個工具的話可以一起來學習一下。謝謝各位看到這邊，如有任何疑問或文章有任何錯漏，歡迎來信與我討論，謝謝！

## 學習資料
1. [TryHackMe/John the Ripper: The Basics](https://tryhackme.com/room/johntheripperbasics) 
2. [John the Ripper - wordlist rules syntax](https://www.openwall.com/john/doc/RULES.shtml)
3. [John the Ripper: The Basics Walkthrough](https://www.youtube.com/watch?v=V405LPqqCCA&t=1784s)

</div>
</body>
</html>
