---
layout: post
title: pentest_tls
feature-img: "assets/img/portfolio/pentest_tls.png"
img: "assets/img/portfolio/pentest_tls.png"
date: 20 May 2025
tags: [Project]
---

# pentest_tls  

Multiple lightweight pentest tools written in Python  

## Installation
- Installed by pip/pip3 (recommended)
```
pip install pentest-tls
```

## Usage
- ***Recommended to use with root privilege***  

### ***pentest_tls IP OPTIONS***  
```
____            _            _     _____           _     
|  _ \ ___ _ __ | |_ ___  ___| |_  |_   _|__   ___ | |___ 
| |_) / _ \ '_ \| __/ _ \/ __| __|   | |/ _ \ / _ \| / __|
|  __/  __/ | | | ||  __/\__ \ |_    | | (_) | (_) | \__ \
|_|   \___|_| |_|\__\___||___/\__|   |_|\___/ \___/|_|___/
                                                          

usage: pentest_tls_cli.py [-h] [-t TYPE] [-p PORTS] [-PU] [-PS] [-PH] [-v] ip

Lightweight Pentest Toolkit

positional arguments:
  ip                    Target IP address

optional arguments:
  -h, --help            show this help message and exit
  -t TYPE, --type TYPE  Tools refer to use. E.g. PS(for port scanner)
  -p PORTS, --ports PORTS
                        Port(s) to scan. E.g. 22,80 or 1-1000
  -PU                   Use UDP ping for port scan
  -PS                   Use TCP SYN packet for port scan. This is the default option
  -PH                   Hybrid, using udp and tcp scan
  -v                    Detailed information for query
```  
  

- **Basic Usage**  

```
pentest_tls 127.0.0.1 -t PS
pentest_tls 127.0.0.1 -t PS -p 80,443
pentest_tls 127.0.0.1 -t PS -p 8000-8080
pentest_tls 127.0.0.1 -t PS -p 80,443,8000-8080
```

- Help Menu
```
pentest_tls --help
pentest_tls -h
```
