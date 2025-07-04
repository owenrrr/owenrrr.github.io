---
layout: post
title:  "Ways to crack the hash"
date:   4 July 2025
categories: Demo
tags: Cybersecurity
---
<html>
<body>
<div markdown="block" style="margin-top: 10px">
    
### Intro
Introduce multiple approaches to crack the hash.
- **john**

    ```bash
    hashid "HASH_YOU_WANT_TO_CRACK"
    john --list=formats | grep HASH_TYPE
    john --format=NT --wordlist=rockyou.txt hash.txt
    ```

- **hashcat**
    - First, go [hashcat examples]() search hash types
    - Then, `hashcat -m 1800 hash.txt rockyou.txt`
    - If you wanna filter out words with specific length, you can do: `awk "length == 8" rockyou.txt > rockyou_l8.txt`


- **Online Tools**
    - **Crackstation**: [Crackstation](https://crackstation.net/)
    - **Hashes.com**: [Hashes.com](https://hashes.com/en/decrypt/hash)
    - **md5decrypt**: [md5decrypt](https://md5decrypt.net/en/#google_vignette)


## 學習資料
1. [Crack The Hash](https://tryhackme.com/room/crackthehash)
2. [Hashcat \| example hashes](https://hashcat.net/wiki/doku.php?id=example_hashes)
3. [Crackstation](https://crackstation.net/)

</div>
</body>
</html>
