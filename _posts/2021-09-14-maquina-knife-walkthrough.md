---
layout: single
title: Máquina Knife - HackTheBox (OSCP Style)
excerpt: "Resolución de la máquina Knife, la cual figura en la plataforma con un nivel de dificultad Easy."
date: 2021-09-14
classes: wide
header:
  teaser: /assets/images/knife-htb/portada.png
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

+ Verificamos que tengamos comunicación a la máquina por medio de una traza ICMP. Por el TTL deducimos que es una máquina Linux.

	![1]
	
+ Iniciamos la fase de reconocimiento con un escaneo rápido con ayuda de nmap en busca de puertos TCP abiertos.

	![2]
	
	Solo vemos dos puertos abiertos, el 22 (SSH) y el 80 (HTTP).
	
+ Corremos un segundo escaneo en busca de versiones y servicios específicos para dichos puertos.

	![3]
	
	No encontramos información que sea relevante.
	
+ Empleando **whatweb** podemos ver algunos detalles técnicos de la web. Destaca la versión de PHP, la cual es una versión **dev**, es decir, no una edición definitiva, sino una de pruebas.

	![4]
	
+ Si entramos al sitio web, no veremos nada de relevancia. Parece ser un sitio web sobre un servicio médico.

	![5]
	
	Su código fuente no contiene información sobre el siguiente paso a seguir.
	
+ Llama la atención la versión de PHP que vimos con **whatweb**, por lo que la buscamos en Exploit Database. Encontramos una backdoor por medio de la cabecera User-Agent.

	![6]
	
	La vulnerabilidad consiste en introducir la cabecera **User-Agentt** (sí, con doble t al final) con la palabra clave **zerodium** seguido de un comando PHP que queramos ejecutar. Si queremos ejecutar un comando en el sistema, podemos usar la instrucción system().
	
	``User-Agentt: zerodiumsystem("whoami");``
	
+ Introduciendo esta cabecera en una petición con **curl**, podemos ver que el comando **whoami** se ejecuta, devolviéndonos el usuario **james**.

	![7]
	
+ Nuestro siguiente objetivo consiste en ejecutar una reverse shell para conseguir acceso al sistema. Se intentó acceder por medio de netcat y por medio de un socket en bash:

	![8]
	
	![9]
	
	Por desgracia, ninguna de estas opciones funcionó.
	
+ Lo siguiente que se intentó es crear un archivo html con un código en bash que entabla un socket para devolvernos una reverse shell, con el objetivo de hacer una petición con curl a dicho recurso y, una vez renderizado, pipearlo con bash para su posterior ejecución.

	![10]

+ Si ponemos un servidor con Python a la escucha con este archivo, y ejecutamos un curl a dicho recurso por medio de la backdoor zerodium, pipeando este curl con bash, podremos ver cómo el sistema lo interpreta y nos devuelve una shell como el usuario james.
	
	![11]
	
	![12]
	
+ Con esta nueva shell, realizamos primero un tratamiento completo de la TTY para trabajar comodamente con esta bash.

	```
	script /dev/null -c bash
	CTRL+Z
	stty raw -echo;fg
			reset
	Terminal type? xterm
	
	export TERM=xterm
	export SHELL=bash
	```
	
	![13]

+ Dentro del directorio del usuario **james**, podremos encontrar la flag de usuario.
	
	![14]
	
+ Si enumeramos el sistema, veremos que en el sudoers está establecido que el usuario **james** puede ejecutar como root, sin proporcionar contraseña, el binario **knife**.
	
	![15]
	
+ De acuerdo con la plataforma GTFObins, para escalar privilegios por medio de este binario, y dentro de un contexto Sudo, debemos ejecutar el siguiente comando:

	``sudo knife exec -E 'exec "/bin/sh"``
	
	![16]
	
+ Ejecutándolo de esta manera, vemos que el sistema nos devuelve una bash como el usuario root.
	
	![17]
	
+ Por último, solo listamos la flag del usuario root dentro del directorio del mismo.
	
	![18]

[0]:/assets/images/knife-htb/0.png
[1]:/assets/images/knife-htb/1.png
[2]:/assets/images/knife-htb/2.png
[3]:/assets/images/knife-htb/3.png
[4]:/assets/images/knife-htb/4.png
[5]:/assets/images/knife-htb/5.png
[6]:/assets/images/knife-htb/6.png
[7]:/assets/images/knife-htb/7.png
[8]:/assets/images/knife-htb/8.png
[9]:/assets/images/knife-htb/9.png
[10]:/assets/images/knife-htb/10.png
[11]:/assets/images/knife-htb/11.png
[12]:/assets/images/knife-htb/12.jpg
[13]:/assets/images/knife-htb/13.png
[14]:/assets/images/knife-htb/14.jpg
[15]:/assets/images/knife-htb/15.png
[16]:/assets/images/knife-htb/16.png
[17]:/assets/images/knife-htb/17.png
[18]:/assets/images/knife-htb/18.jpg