---
layout: single
title: Máquina Active - HackTheBox (OSCP Style)
excerpt: "Resolución de la máquina Active, la cual figura en la plataforma con un nivel de dificultad Easy."
date: 2021-09-14
classes: wide
header:
  teaser: /assets/images/active-htb/portada.png
  teaser_home_page: true
categories:
  - HTB
  - Blog
tags:
  - Blog
  - Writeup
  - Guía
  - Walkthrough
---

+ **Información de la máquina**
	
	![0]

+ Verificamos que tengamos comunicación a la máquina por medio de una traza ICMP. Por el TTL deducimos que es una máquina Windows.

	![1]
	
+ Iniciamos la fase de reconocimiento con un escaneo rápido con ayuda de nmap en busca de puertos TCP abiertos.

	![2]
	
	Vemos muchos puertos habilitados, pero realtan el 88 (Kerberos), 389 (ldap) entre otros que son característicos de un Domain Controller en un entorno de Active Directory.
	
+ Enumeramos el servicio SMB con **crackmapexec**. Vemos que, efectivamente, se trata de un Domain Controller cuyo dominio es **active.htb**.

	![3]
	
	Podría aplicarse virtual hosting del lado del servidor.
	
+ Para corroborar esta última teoría, añadimos el nombre de dominio al /etc/hosts

	![4]
	
+ Al enfrentarnos a un Domain Controller, primeramente necesitamos de un listado de usuarios válios a nivel de dominio, esto con el objetivo de ejecutar un ataque de tipo asreproast o kerberoasting. Para ello, primero intentamos entrar enumerar por medio de **rpcclient** empleando una null session.
	
	``rpcclient 10.10.10.100 -N``

	![5]
	
	Por desgracia, vemos que el sistema no nos lo permite.
	
+ Como el servicio SMB se encuentra habilitado, enumeramos los recursos compartidos a nivel de red con **smbmap** por medio de una null session (recordemos, aún no contamos con credenciales válidas).

	``smbmap -H 10.10.10.100 -u ""``
	
	![6]
	
	Visualizamos una carpeta dentro del recurso, llamada **Replication**.
	
+ Si entramos a este recurso, veremos una carpeta llamada **active.htb**

	``smbmap -H 10.10.10.100 -u "" -r Replication``
	
	![7]
	
	Nuestro siguiente paso será enumerar todo este directorio en busca de información relevante.
	
+ Después de indagar en varias carpetas, nos encontramos con un archivo llamado **Groups.xml**

	![8]
	
	Este archivo es muy importante, ya que contiene credenciales de usuario con una contraseña encriptada, la cual podemos descifrar fácilmente.
	
	Y TODO GRACIAS A MICROSOFT
	
+ Hace un par de años, Microsoft publicó la clave AES con la cual se podían encriptar estas contraseñas almacenadas en archivos del Domain Controller.

	![9]
	
	Como podemos ver dentro del archivo **Groups.xml**, contamos con un usuario llamado **SVC_TGS** y una contraseña que no está en plain text, está encriptada.
	
	![10]
	
	Para desencriptarla, solo debemos usar la herramienta gpp-decrypt, la cual incluye la clave proporcionada por Microsoft para realizar esta tarea.

+ Por medio de gpp-decript, desciframos esta contraseña para poder verla en plain text.
	
	![11]
	
+ Ya teniendo unas credenciales válidas, podemos comenzar de nueva cuenta con nuestra enumeración inicial. Primero, probemos a conectarnos nuevamente por medio de rpcclient para tratar de enumerar usuarios del dominio.
	
	![12]
	
	Vemos que en esta ocasión podemos acceder al recurso y enumerar usuarios, por lo que los exportamos a un archivo de texto mediante el cual podamos ejecutar un ataque asreproast en busca de un Ticket Granting Ticket (TGT), que no es más que un hash que se puede crackear de manera online con el objetivo de obtener nuevas credenciales (si es que hay).

+ Si intentamos correr un ataque asreproast, veremos que no somos capaces de obtener un TGT, por lo que este ataque no puede realizarse dentro de este contexto.

	``GetNPUsers.py active.htb/ -no-pass -usersfile users``
	
	Donde users es el archivo que contiene a todos los usuarios enumerados con **rpcclient**
	
	![13]
	
+ Hay algo de lo que no nos hemos percatado, y es el nombre del usuario del que disponemos credenciales. Un ataque kerberoasting tiene como objetivo obtener un Ticket Granting Service (TGS), que al igual que el TGT, es un hash que se puede crackear de manera offline si el sistema cuenta con usuarios kerberoasteables. Esto de TGS llama la atención, dado que es parte del nombre de usuario del que disponemos.
	
	Por lo tanto, procedemos a intentar un ataque kerberoasting, el cual requiere de unas credenciales válidas para poderse efectuar. Se las proporcionamos.
	
	``GetUsersSPNs.py`` active.htb/SVC_TGS:PASSWORD``
	
	![14]
	
	Como podemos observar, el sistema cuenta con un usuario kerberoasteable, y es nada más y nada menos que el mismísimo usuario Administrator. Lo siguiente sería intentar obtener su ticket TGS para crackearlo offline y obtener sus credenciales de acceso.
	
+ Para obtener su TGS, basta con añadir la bandera -request al final del comando ejecutado anteriormente.

	``GetUsersSPNs.py`` active.htb/SVC_TGS:PASSWORD -request``
	
	![15]
	
	Vemos que el programa devuelve un error: **"Clock skew too great"**. Este error es ocasionado porque el reloj de nuestro sistema no está sincronizado con el reloj del Domain Controller.
	
+ Para sincronizar nuestro reloj con el del DC, ejecutamos la herramienta **ntpdate**:
	
	``ntpdate active.htb``
	
	Posteriormente, ejecutamos de nueva cuenta el request del TGS para el usuario Administrator, y vemos cómo, en esta ocasión, el programa se ejecuta correctamente, pudiendo visualizar ahora el hash.
	
	![16]
	
+ Este hash, como se mencionó anteriormente, se puede crackear por medio de fuerza bruta con ayuda de John.
	
	![17]
	
	La contraseña es ``Ticketmaster1968``
	
+ Validamos estas credenciales con **crackmapexec** para el servicio SMB (dado que WinRM no se encuentra habilitado).
	
	![18]
	
	Vemos que la consulta devuelve un **Pwned!**, lo que significa que tenemos privilegios de psexec para acceder directamente al sistema como este usuario.
	
+ Por medio de psexec, entablamos una conexión con el DC empleando las credenciales que acabamos de obtener.
	
	![19]
	
	El sistema nos permite el acceso como el usuario administrador, por lo que podemos afirmar que hemos comprometido por completo la máquina.

+ Solo restaría enumerar las flags de usuario y de root del sistema, las cuales se encuentran en sus respectivos directorios.
	
	![20]

[0]:/assets/images/active-htb/0.png
[1]:/assets/images/active-htb/1.png
[2]:/assets/images/active-htb/2.png
[3]:/assets/images/active-htb/3.png
[4]:/assets/images/active-htb/4.png
[5]:/assets/images/active-htb/5.png
[6]:/assets/images/active-htb/6.png
[7]:/assets/images/active-htb/7.png
[8]:/assets/images/active-htb/8.jpg
[9]:/assets/images/active-htb/9.png
[10]:/assets/images/active-htb/10.jpg
[11]:/assets/images/active-htb/11.png
[12]:/assets/images/active-htb/12.png
[13]:/assets/images/active-htb/13.png
[14]:/assets/images/active-htb/14.png
[15]:/assets/images/active-htb/15.png
[16]:/assets/images/active-htb/16.png
[17]:/assets/images/active-htb/17.png
[18]:/assets/images/active-htb/18.png
[19]:/assets/images/active-htb/19.png
[20]:/assets/images/active-htb/20.jpg