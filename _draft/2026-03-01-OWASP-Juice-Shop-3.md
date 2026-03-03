---
layout: post
title:  "OWASP Juice Shop Walkthrough (1)"
date:   2026-03-01 11:15:55 +0800
categories: Demo
tags: OWASP-Juice-Shop
pinned: true
---
<html>
<body>
<div markdown="block" style="margin-top: 10px">
    
## Introduction:
In this writeup, we will start from the beginning of pwning OWASP Juice Shop to a comprehensive walkthrough. Let's don't talk too much and get started! Here's the OWASP Juice Shop official document: [OWASP Juice Shop](https://pwning.owasp-juice.shop/companion-guide/latest/index.html).
This article includes all three star challenges!

<br><br>

## Outline

| Challenge | Vulnerability |
| --- | --- |
| [3.1 Forged Feedback](#forged-feedback) | Broken Access Control |
| [3.2 Login Jim](#login-jim) | SQL Injection |
| [3.3 Login Bender](#login-bender) | SQL Injection | 
| [3.4 Admin Registration](#admin-registration) | Improper Input Validation | 
| [3.5 Bjoern's Favorite Pet](#bjoerns-favorite-pet) | Broken Authentication | 
| [3.6 CAPTCHA Bypass](#captcha-bypass) | Broken Anti Automation |  
| [Exposed Metrics](#17-exposed-metrics) | Observability Failures |  
| [Mass Dispel](#18-mass-dispel) | Misc. |  
| [Missing Encoding](#19-missing-encoding) | Improper Input Validation |  
| [Outdated Allowlist](#110-outdated-allowlist) | Unvalidated Redirects |  
| [Repetitive Registration](#111-repetitive-registration) | Improper Input Validation |  
| [Web3 Sandbox](#112-web3-sandbox) | Broken Access Control |  
| [Zero Stars](#113-zero-stars) | Improper Input Validation |  


<br><br>

## 3 Star Challenge

### 3.1 Forged Feedback
```
Post some feedback in another user's name.
```

Go to Customer Feedback page (`/contact`). Open the proxy and use Burp Suite to intercept the feedback request. The request body contains a JSON form data which includes `userId`, `captchaID`, `captcha`, `comment` and `rating` keys. We could modify the `userId` value to another user's ID (In this challenge, we don't have to know actual another user's ID. Just modify `userId` to different number).

```json
{"UserId":19,"captchaId":1,"captcha":"-31","comment":"hi (***a@juice-sh.op)","rating":2}
```

- **Root Cause**
The application trusts a client-controlled `UserId` field when submitting feedback. Because the backend accepts the user identity from the request instead of deriving it from the authenticated session/token, an attacker can modify the `UserId` and submit feedback on behalf of another user.

- **Remediation**
Remove any client-side control over ownership fields such as `UserId`. On the server side, derive the feedback owner exclusively from the authenticated user context, enforce authorization checks before creating records, and treat unauthenticated submissions as anonymous only.

<br>

### 3.2 Login Jim
```
Log in with Jim's user account.
```

As long as we know that SQLi workds in the login function(2.2 Login Admin), and we already got Jim's email address which appears in multiple places (e.g. through admin's portal you could grab all user's email address, or through product reviews, or simply take a guess). Let's try a common SQLi technique.

```
email:jim@juice-sh.op' --
password: 12345
```

We could also grab detailed SQL related information by crafting error SQL syntax through the login function. Let's try `jim@juice-sh.op'` and see what happens (Note that we could only see `[object Object]` on browser because of Javascript, `Object.prototype.toString()` which will translate the JSON object to string resulting in `[object Object]`. I grab the following JSON object through Burp Suite.) This contains rich information. The website is using SQLite as database engine. There's a table named `Users` which includes columns `email` and `password`.

```json
{
  "error": {
    "message": "SQLITE_ERROR: unrecognized token: \"827ccb0eea8a706c4c34a16891f84e7b\"",
    "stack": "Error\n    at Database.<anonymous> (/juice-shop/node_modules/sequelize/lib/dialects/sqlite/query.js:185:27)\n    at /juice-shop/node_modules/sequelize/lib/dialects/sqlite/query.js:183:50\n    at new Promise (<anonymous>)\n    at Query.run (/juice-shop/node_modules/sequelize/lib/dialects/sqlite/query.js:183:12)\n    at /juice-shop/node_modules/sequelize/lib/sequelize.js:315:28\n    at process.processTicksAndRejections (node:internal/process/task_queues:105:5)",
    "name": "SequelizeDatabaseError",
    "parent": {
      "errno": 1,
      "code": "SQLITE_ERROR",
      "sql": "SELECT * FROM Users WHERE email = 'jim@juice-sh.op'' AND password = '827ccb0eea8a706c4c34a16891f84e7b' AND deletedAt IS NULL"
    },
    "original": {
      "errno": 1,
      "code": "SQLITE_ERROR",
      "sql": "SELECT * FROM Users WHERE email = 'jim@juice-sh.op'' AND password = '827ccb0eea8a706c4c34a16891f84e7b' AND deletedAt IS NULL"
    },
    "sql": "SELECT * FROM Users WHERE email = 'jim@juice-sh.op'' AND password = '827ccb0eea8a706c4c34a16891f84e7b' AND deletedAt IS NULL",
    "parameters": {}
  }
}
```

- **Root Cause**
The login function is vulnerable to SQL injection because it inserts the user-supplied email directly into the SQL authentication query. This allows an attacker to use a payload such as `jim@juice-sh.op'--` to comment out the password condition and authenticate as Jim. The application further worsens the issue by exposing raw SQL errors to the client.

- **Remediation**
Replace dynamic SQL string construction with parameterized queries/prepared statements (Java, C#, .Net .etc), return only generic authentication errors to users, and avoid exposing backend SQL details in responses.

<br>

### 3.3 Login Bender
Same as [3.2 Login Jim](#login-jim)

<br>

### 3.4 Admin Registration
```
Register as a user with administrator privileges.
```

We first try to register a new user and see what the request body and response body is. The following is the original request body:

```json
{"email":"demo3@juice-sh.op","password":"12345","passwordRepeat":"12345",
"securityQuestion":{"id":1,"question":"Your eldest siblings middle name?","createdAt":"2026-03-01T16:44:31.062Z","updatedAt":"2026-03-01T16:44:31.062Z"},"securityAnswer":"12345"}
```

We observed that there's an interesting key in the response body `"role": "customer"`. However, we don't have it in the request body. 

```json
{"status":"success","data":{"username":"","deluxeToken":"","lastLoginIp":"0.0.0.0","profileImage":"/assets/public/images/uploads/defaultAdmin.png","isActive":true,"id":27,"email":"demo3@juice-sh.op","role":"customer","updatedAt":"2026-03-01T18:39:33.108Z","createdAt":"2026-03-01T18:39:33.108Z","deletedAt":null}}
```

Let's take a guess and try to add this key-value pair into the request body and see what happens. As long as we found that there's no error generated by doing that, we could take a further guess with `"role": "administrator"` or `"role": "admin"`. The one with `"role": "admin"` seems to be correct.

```json
{"email":"demo3@juice-sh.op","role": "admin","password":"12345","passwordRepeat":"12345",
"securityQuestion":{"id":1,"question":"Your eldest siblings middle name?","createdAt":"2026-03-01T16:44:31.062Z","updatedAt":"2026-03-01T16:44:31.062Z"},"securityAnswer":"12345"}
```

- **Root Cause**
The registration endpoint trusts a client-controlled `role` field. Because the backend accepts and binds sensitive properties from the request body during user creation, an attacker can submit `"role": "admin"` and create an account with administrative privileges. This is a classic mass assignment / over-posting issue caused by improper input validation.

- **Remediation**
Remove `role` from all public registration input handling, force new self-registered accounts to receive a safe default role on the server side, and restrict privileged account creation to a protected administrative workflow. Implement strict allow-listing for bindable registration fields and use DTOs so sensitive properties such as `role` cannot be set by client input.

<br>

### 3.5 Bjoern's Favorite Pet
```
Reset the password of Bjoern's OWASP account via the Forgot Password mechanism with the original answer to his security question.
```

There're several accounts related to Bjoern: `bjoern.kimminich@gmail.com`, `bjoern@juice-sh.op` and `bjoern@owasp.org`. We can find them through either admin portal `/administration` or via product reviews. We should target the one account which security questions is "Name of your favorite pet". The account `bjoern@owasp.org` seems to be the one. First I tried `kimminich` as the answer and it failed. I read through every product review with `bjoern@owasp.org` and every photo's dscription and nothing interesting. Finally I searched online with Bjoern's cat. There are many OWASP walkthrough which can't identify as the correct solution. After that, I found a tweet posted by Bjoern which said *Zaya-the-three-legged-cat...* with a picture of a cat. **Zaya** is the correct answer.

- **Root Cause**
The application relies on a knowledge-based security question as the primary verifier for password reset. In this challenge, Bjoern’s answer is truthful and discoverable through public information (OSINT), so an attacker can use that answer to reset his password and take over the account. This is a weak account recovery design, not a code-level bug. *The knowledge-based authentication（KBA）is no longer recognized as an acceptable authenticato* -- [NIST](https://pages.nist.gov/800-63-FAQ/).

- **Remediation**
Remove security questions from self-service password reset. Replace them with stronger recovery mechanisms such as protected recovery factors, one-time recovery codes, device-backed out-of-band verification, or identity re-proofing when necessary. Add rate limiting and generic responses to reduce guessing and account enumeration risk.

<br>

### 3.6 CAPTCHA Bypass
```
Submit 10 or more customer feedbacks within 20 seconds.
```



<br><br>

## Materials
1. [OWASP Juice Shop Official Document](https://pwning.owasp-juice.shop/companion-guide/latest/index.html)
2. [OWASP SQLi Prevention Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/SQL_Injection_Prevention_Cheat_Sheet.html?utm_source=chatgpt.com)
3. [NIST KBA security](https://pages.nist.gov/800-63-FAQ/)

<br><br>

## Next Article
[OWASP Juice Shop Walkthrough (2)](https://owenrrr.github.io/demo/2026/02/27/OWASP-Juice-Shop-2.html)

</div>
</body>
</html>
