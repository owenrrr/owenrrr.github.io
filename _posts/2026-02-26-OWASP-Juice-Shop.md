---
layout: post
title:  "OWASP Juice Shop Walkthrough"
date:   2025-02-26 11:15:55 +0800
categories: Demo
tags: Cybersecurity Writeup
pinned: true
---
<html>
<body>
<div markdown="block" style="margin-top: 10px">
    
### Introduction:
In this writeup, we will start from the beginning of pwning OWASP Juice Shop to a comprehensive walkthrough. Let's don't talk too much and get started! Here's the OWASP Juice Shop official document: [OWASP Juice Shop](https://pwning.owasp-juice.shop/companion-guide/latest/index.html).
  
### Outline
- [Installation](#installationdocker)
- [Score Board](#score-board)
- [1-Star Challenge](#1-star-challenge)

<br><br>

### Installation(Docker)
There're multiple ways to install juice shop on your machine. I choose docker because it's already installed on my machine. Furthermore, docker is isolated in the environment so it won't affect the existing programs or services.

1. Download the latest docker image
```bash
docker pull bkimminich/juice-shop
```

2. Launch the container
```bash
docker run -d -p 127.0.0.1:3000:3000 bkimminich/juice-shop
```

3. Quick Test
```bash
curl -I http://localhost:3000
```

<br><br>

### Score Board
There's a scoreboard provided by Juice Shop which is hidden in the URL. We can access it through adding `score-board` to the URL and find out how many vaulnerabilities are there in this vulerable website. It also provides learners a decent way to record how many vulnerabilities are left in the website. The challenges are rated with a difficulty level between 1-star to 6-star. We will likely to complete it from the easist to the most difficult ones.
![OWASP Juice Shop Scoreboard](https://pwning.owasp-juice.shop/companion-guide/latest/_images/part1/score-board_partly.png)

<br><br>

### 1-Star Challenge

#### 1.1 DOM XSS
```
Perform a DOM XSS attack with <iframe src="javascript:alert(`xss`)">
```

After I browse the website for a while, I found there's a direct way to insert this DOM XSS that is the search box. For this vulnerability, just directly type `<iframe src="javascript:alert('xss')">` in the searchbox.

#### 1.2 Bonus Payload
```
Use the bonus payload <iframe width="100%" height="166" scrolling="no" frameborder="no" allow="autoplay" src="https://w.soundcloud.com/player/?url=https%3A//api.soundcloud.com/tracks/771984076&color=%23ff5500&auto_play=true&hide_related=false&show_comments=true&show_user=true&show_reposts=false&show_teaser=true"></iframe> in the DOM XSS challenge.
```

We could take a closer look at this payload. It's supposed to be a video/audio recording due to `allow="autoplay"` iframe attribute.

#### 1.3 Privacy Policy
```
Read our privacy policy.
```

Just Register an account and read privacy policy.

#### 1.4 Bully Chatbot
```
Receive a coupon code from the support chatbot.
```

Just simply communicate with chatbot and try to bypass it. There're at least two responses from the chatbot to get the coupon. The first response is "You should check out our social media channel for monthly coupon..." and we could get the coupon from Juice Shop Reddit which we could find on About Me page. The second would be "Oooookay, if you promise to stop nagging me here's a 10% coupon code for you: XXXXXXX".

#### 1.5 Confidential Document
```
Access a confidential document
```

The entry-point of this vulnerability is About me page. There's a link which directs me to `http://localhost:3000/ftp/legal.md`. Now, we should wonder what files are under `ftp/` directory as long as we know that FTP is a file management service for managing files. After accessing `ftp/` we found that there're several files and one folder named `quarantine`. Just simply viewing some files.

#### 1.6 Error Handling
```
Provoke an error that is neither very gracefully nor consistently handled.
```

If we try out the way on previous steps viewing files under `ftp/` directory, we may provoke a 403 Error with an error message: "Only .md and .pdf files are allowed!".

#### 1.7 Exposed Metrics
```
Find the endpoint that serves usage data to be scraped by a popular monitoring system.
p.s popular monitoring system: prometheus
```

Prometheus is a systems and monitoring system. This type of project requires configuration settings including paths. We can search for default endpoints/paths for sering usage data. We found that `./data` for data storage and `/metrics` for HTTP metric endpoint. Try `/metrics`.

#### 1.8 Mass Dispel
```
Close multiple "Challenge solved"-notifications in one go.
```

On the [official document](https://pwning.owasp-juice.shop/companion-guide/latest/part1/challenges.html), it says - *you can simply Shift-click one of their X-buttons to dismiss all at the same time.*

#### 1.9 Missing Encoding
```
Retrieve the photo of Bjoern's cat in "melee combat-mode".
```

Speak to photo, we found there's a Photo Wall on the side navigation bar. As we viewed the Photo Wall page, we found the first image is invalid due to its src attribute. It contains some invalid characters so that the interpreter can not recognize the URL. We could URLEncode it and replace it afterwards `%E1%93%9A%E1%98%8F%E1%97%A2-%23zatschi-%23whoneedsfourlegs-1572600969477.jpg`. Note that the prefix cannot be modified `assets/public/images/uploads/`. Otherwise the scoreboard can't detect the success. (However, it will work)

#### 1.10 Outdated Allowlist
```
Let us redirect you to one of our crypto currency addresses which are not promoted any longer.
```

Most of these code will be written in JS file. We simply do a check on all JS files. Typed a keyword "redirect" which seems to appear in the redirect URL and found that there're three JS files containing redirect keyword: `main.js`, `989.js` and `vendor.js`. After a quick glimpse, only `main.js` contains some suspicious and valid URL which formats like `url='./redirect?to=xxx'`. We simply try all of them(3) and found the one contains `https://blockchain.com` is the correct one.

#### 1.11 Repetitive registeration
```
Follow the DRY principle while registering a user.
```

The description leads us to the Registration page. We got an email input, a password and repeat password input, and a security question and corresponding answer. As a vulnerability, we need to break the validation rule which is obvious the password and its confirmation input. My solution is to make sure the Register button isn't disabled when you don't repeat your password intentionally. In disabled button, we need to remove `mat-mdc-button-disabled` class and `disabled="true"` attribute. By doing this, the Register button would become available.

There's also an easier way to achieve this challenge. Just use burp suite, catch the request and modify the repeat password (`repeatPassword`) value, finally send the request.

#### 1.12 Web3 Sandbox
```
Find an accidentally deployed code sandbox for writing smart contracts on the fly.
```

We searched Web3 Sandbox default path in Google and found that the path is generally `/web3-sandbox`.

#### 1.13 Zero Stars
```
Give a devastating zero-star feedback to the store.
```

Same as challenge [1.11 Repetitive Registration](#111-repetitive-registeration1.11 Repetitive registeration). Use burp suite to catch the request and modify the rating value.


## Materials
1. [OWASP Juice Shop Official Document](https://pwning.owasp-juice.shop/companion-guide/latest/index.html)

</div>
</body>
</html>
