---
layout: post
title:  "Passive Reconnaissance (1)"
date:   2025-04-30 21:00:55 +0800
categories: Demo
tags: Cybersecurity
---
<html>
<body>
<div markdown="block" style="margin-top: 10px">
    
### Intro
本篇來介紹幾個常用的Passive Reconnaissance工具來供大家參考，並提供相關的背景知識和基本使用方法，其中包含：
- whois
- nslookup
- dig
- DNSDumpster
- Shodan  
而介紹工具前必須先瞭解此工具的用處、背景再來才是工具的基本用法，這樣才能理解透徹並善加使用  


### whois
- **用處**： 查看此domain註冊時提供的信息，注意，由於安全意識抬頭且此工具被大家熟知，故而可以在此紀錄上做手腳，因此用此工具獲得的資訊必須妥善使用
- **背景知識**： 每一位開發者/公司在向Registrar註冊網域名稱時都會提供相應資訊，如公司註冊人、電話、郵箱等。Registrar再向Whois Server提供資料已作後續查詢或其他用途。而我們使用的whois工具中查詢的便是多個Whois server組成的集合
- **基本使用**：
```
whois google.com
```

  
### nslookup
- **用處**： 它可以幫助你確認一個網域名稱對應的 IP 地址、DNS 記錄的內容、以及測試 DNS 是否正確運作
- **背景知識**： 首先要先瞭解DNS是什麼?

*The Domain Name System (DNS) is a hierarchical and distributed name service that provides a naming system for computers, services, and other resources on the Internet or other Internet Protocol (IP) networks* 
  
簡單來說，DNS就是幫你把google.com轉換成ipv4或ipv6的地址。DNS Server就是儲存這個對照關係的數據庫，被每個Registrar保存並用於在有人使用網域名稱是給予對照的地址。值得一提的是，這個數據庫中存著的數據對照關係不單單只是前面提到的網域名稱對映ipv4/ipv6地址，以下是多個 *DNS Record*：  
    - A: 網域名稱->IPv4  
    - AAAA: 網域名稱->IPv6  
    - CNAME: 網域別名->網域名稱  
    - MX: 網域名稱->Mail Server  
    - TXT: 儲存任意資料  
    - SOA: 此域名設置  

- **基本使用**：  
***nslookup OPTIONS DOMAIN_NAME SERVER***   

```
OPTIONS(要查詢的DNS Record類別)
  -type=A
DOMAIN_NAME(要查詢的域名)
  google.com
SERVER(可選，要指定查詢的DNS Server)
  1.1.1.1

nslookup -type=A google.com 1.1.1.1
nslookup -type=MX google.com
```  
  

### dig(Domain Information Groper)
- **用處**： 與nslookup使用場景基本類似，但提供更多資訊
- **基本使用**：  
***dig @SERVER DOMAIN_NAME TYPE***  

```
SERVER(可選，要指定查詢的DNS Server)
  1.1.1.1
DOMAIN_NAME(要查詢的域名)
  google.com
TYPE(要查詢的DNS Record類別)
  TXT

dig google.com TXT
dig @1.1.1.1 google.com MX
```
 

### DNSDumpster
- **用處**： 是一個網站工具，可以通過瀏覽器使用非常方便。他與 `nslookup` 和 `dig` 工具不同的地方是， `DNSDumpster` 可以查詢到子域名，例如你查詢 `google.com` 可以搜尋到 `admin.google.com` 等等子域名，對於要搜集有關網域資料的人來說非常有用 
- **基本使用**：
直接打開瀏覽器搜尋 [DNSDumpster](https://dnsdumpster.com/) 或直接點擊連結即可。搜尋容易功能強大，就是免費版對結果數量有限制，沒辦法查找到最完全的資訊(但也非常夠用)
  
   

### Shodan
- **用處**： 是一個更加強大且全面的網站工具，它注重的是所有與網路連接的設備檢測（包含IoT等），故而非常強大。也可用在DNS搜索、OS/Server版本型號檢測等。`Shodan` 比較像是查看統計數據的工具，以及該連接設備的詳細資料，而並非只著重在DNS Query
- **基本使用**：
直接打開瀏覽器搜尋 [Shodan](https://www.shodan.io/) 或直接點擊連結即可。可以搜尋域名 `google.com`、OS版本 `Ubuntu`、Server版本 `Apache/2.4.18`等等，功能非常全面  
  
  

## 學習資料
1. TryHackMe CompTIA Pentest+ / Information Gathering and Vulnerability Scanning
Passive Reconnaissance
2. [DNSDumpster](https://dnsdumpster.com/) 
3. [Shodan](https://www.shodan.io/)

</div>
</body>
</html>
