---
layout: post
title: PTSPSeeker
feature-img: "assets/img/portfolio/ptsp_1.png"
img: "assets/img/portfolio/ptsp_1.png"
date: 1 July 2025
tags: [Project]
---
# Intro
This is the project that provides a lightweight search script for searching related penetration testing scripts, commands, and urls. You can visit the project on my [github repository](https://github.com/owenrrr/PTSPseeker).

# PTSPseeker
PTSP stands for PenTest ScriPts. It's used for searching a variety of pentest scripts/commands/links. It's outstanded for its simplicity, flexibility, and lightweighted.

## Basic Usage

```bash
# Method 1
chmod +x seeker.sh
./seeker.sh

# Method 2
bash seeker.sh
```

## Update Database
PTSPSeeker accepts users to customize the script and database. If you want to add some new scripts and update the seeker.db file, just run update.sh file after adding files to database folder. Currently I've not completed the functionality of utilizing module type(file,cmd,url). However, this feature will definitely be available in the future.

You can just simply run update.sh and it'll automaticlly detect files under database folder and modifies seeker.db file.

```bash
# Method 1
chmod +x update.sh
./update.sh

# Method 2
bash update.sh
```

## Demo
- Simple Usage
![p1](/assets/img/portfolio/ptsp_1.png)


- Update Progress
![p2](/assets/img/portfolio/ptsp_2.png)
