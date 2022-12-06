---
layout: single
title: Neighbour Machine - TryHackMe
excerpt: "This is a very easy machine for beginners. However, we are going to resolve it step by step, learning about code review and the IDOR vulnerability."
date: 2022-12-06
classes: wide
header:
  teaser: /assets/images/neighbour/portada.png
  teaser_home_page: true
categories:
  - Comunidad
  - Blog
  - Writeup
tags:
  - code
  - idor
---

# Introduction
The Neighbor machinecan be found at [TryHackMe](https://tryhackme.com/room/neighbour). It is a very easy machine for those people who just entered the world of cybersecurity and that allows them to understand concepts such as code review and what an IDOR is.

We will solve the room step by step until obtaining the requested flag, explaining in detail each aspect of the process for those who are just starting out in this world.

It is a room where there is only one web application that we must compromise in order to obtain the flag, so there will be no recognition or scanning stages.

# Index
- [Introduction](#introduction)
- [Index](#index)
- [Web service](#web-service)
- [Reviewing source code](#reviewing-source-code)
- [IDOR (Insecure Direct Object Reference)](#idor-insecure-direct-object-reference)

Web service
==================================================================================================================
TryHackMe tells us that we should go to the website http://10.10.151.197, there, we can see a login portal as shown above:

![1]

Reviewing source code
==================================================================================================================
As we can see, the same portal tells us that if we want to log in as a **guest** user, we must type the **Ctrl+U** shortcut. This shortcut is used to view the source code of the website, something that should always be reviewed when carrying out an assessment of a website, as developers can often leave relevant information about the service in comments.

When reviewing the source code, we found the following:

![2]

Near the end of the source code, we see a comment that includes the **guest** user's credentials and information about another user on the system called **admin**.

If we login with guest's credentials, we can see how the website welcomes us:

![3]

IDOR (Insecure Direct Object Reference)
==================================================================================================================
When doing an assessment of a web application, something else to check is how information is sent to the site through parameters in the URL. Notice how the URL has the following format:

    http://10.10.151.197/profile.php?user=guest

As we can see, the name of the logged in user is being sent through the user parameter in the URL. What would happen if we modified this value to another user we already know? Remember that we had already seen another user called **admin**.

    http://10.10.151.197/profile.php?user=admin

![4]

Now the web application thinks we are the **admin** user and we are able to retrieve the flag. What just happened is that we have taken advantage of a vulnerability known as **IDOR**, which is a type of access control vulnerability.

This can occur when a web server receives user-supplied input to retrieve objects (like files, data, documents), and it is not validated on the server-side to confirm the requested object belongs to the user requesting it. In this case, we have requested access as another user, and on the server-side it is not validated that we really have the permissions or that we have previously authenticated as that user.

[1]:/assets/images/neighbor/1.png
[2]:/assets/images/neighbor/2.png
[3]:/assets/images/neighbor/3.png
[4]:/assets/images/neighbor/4.png