---
layout: single
title: Máquina Cap - HackTheBox (OSCP Style)
excerpt: "Resolución de la máquina Cap la cual figura en la plataforma con un nivel de dificultad Easy."
date: 2021-10-02
classes: wide
header:
  teaser: /assets/images/cap-htb/portada.png
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

# Información de la máquina

La máquina figura en Hack The Box como una máquina Linux. Su dificultad es Easy, de 20 puntos.

![0]

# Reconocimiento
Por medio de un escaneo de puertos con nmap, detectamos aquellos que se encuentren activos con un estatus **open** en la máquina víctima:

![1]

Vemos que se encuentran habilitados únicamente el 21 (FTP), 22 (SSH) y 80 (HTTP).

Si realizamos un escaneo más exhaustivo en busca de servicios y versiones específicas de dichos puertos, nos encontramos con una gran cantidad de información:

	nmap -sC -sV -p21,22,80 10.10.10.245 -oN targeted
    
![2]

En el caso del servicio HTTP, el escaneo devuelve una cantidad considerable de información. Sin embargo, esta es irrelevante.

# Servicio FTP
Si intentamos ingresar al servicio FTP por medio de una sesión como anónimo, veremos que el sistema no nos lo permite. Es decir, necesitamos credenciales váidas para acceder.

![3]

Volveremos a este servicio más adelante.

# Servicio HTTP
Si entramos al sitio web por medio de un navegador, nos encontraremos con la siguiente interfaz:

![4]

Del lado izquierdo veremos varias funciones de la página. Si abrimos una por una veremos una serie de páginas con contenido interesante.

En primer lugar, la función **IP Config** despliega en pantalla la salida del comando **ifconfig**.

![5]

la siguiente función es **Network Status**, la cual despliega la salida del comando **netstat**.

![6]

Al parecer, estos comandos son ejecutados en tiempo real en la máquina víctima al momento de consultarlos por medio del sitio web. Este razonamiento me llevo a cuestionarme si en la petición **HTTP** no viajaría el comando que se esté ejecutando y, de esta manera, efectuar un code injection. Inbtercepté la petición por medio de **Burpsuite**.

![7]

Por desgracia, como podemos observar, la petición solo realiza un GET a una página en concreto sin llevar una petición de por medio, así que intuyo que en el servidor está configurado que, en caso de solicitar dicha página, el comando ya definido sea ejecutado.

# Archivos CAP
Enumerando aún más, nos encontramos con la última función de la página, llamada **Security Snapshot**. Según su descripción, ejecuta un pcap del tráfico de red que suceda en ese momento, y devuelve un análisis de la captura obtenida. Para ponerla a prueba, se ejecutó un **ping** a la máquina mientras esta captura de paquetes se llevaba a cabo, obteniendo el siguiente resultado:

![8]

La captura se puede descargar; si la analizamos con **Wireshark**, veremos que efectivamente, capturó las trazas ICMP del comando ping que enviamos.

![9]

Si embargo, poco podemos hacer con esto. Al analizar más la página web, veo que en la URL solicitó el archivo 1 cuando hice esta captura desde la URL. Por lo que intenté modificar el parámetro por otro número, encontrándome así con la captura 0, la cual contiene mucha información.

![10]

Si analizamos esta captura nuevamente con Wireshark, veremos un montón de peticiones **HTTP** y **TCP**.

![11]

Si hacemos un Follow del TCP Stream del primer paquete HTTP, veremos información acerca de una petición al sitio web. El punto de analizar una captura, es analizar los diferentes streams de la petición. Al avanzar, llegaremos a un stream que contiene las credenciales del servicio FTP.

![12]

# De nuevo al FTP
Si intentamos acceder al FTP con estas nuevas credenciales, veremos que son válidas y ya podemos enumerar el servicio.

Vemos un directorio que contiene los archivos de un directorio de usuario en Linux; incluso podemos ver la flag de usuario.

![13]

Si intentamos descargar la flag, vemos que el sistema nos lo permite. Sin embargo, tenemos prohibido la subida de archivos al servidor, por lo que poco podemos hacer en este punto, dado que ningún directorio contiene información que sea relevante para continuar.

![14]

Mientras tanto, ya tenemos la flag de usuario:

![15]

# SSH
Dado que tenemos credenciales, y un servicio restante por examinar, probamos estas en el SSH para ver si podemos acceder. Vemos que las credenciales son válidas, y ya tenemos acceso a la máquina.

![16]

# Escalada de Privilegios
Dado que ya tenemos la flag de usuario, lo que resta es escalar privilegios en la máquina. Iniciamos con una búsqueda de archivos **SUID**.

	find / -perm -u=s 2>/dev/null
    
![17]

Por desgracia, esta búsqueda no devuelve ningún binario del que podamos abusar para escalar privilegios. Lo siguiente será buscar **capabilities** en el sistema:

	getcap -r / 2>/dev/null

Vemos que este escaneo devuelve un binario con una **capability** de la que podemos abusar para escalar privilegios: El binario **python3.8** y la capability **cap_setuid**.

![18]

Esta capability lo que nos permitirá es hacer un cambio de UID a 0 (es decir, usuario **root**), solo si este último es el propietario del binario. Posteriormente, solo tendremos que spawnear una bash y seremos usuario **root**.

Listamos al propietario del binario:

![19]

Vemos que efectivamente, root es el propietario del binario, por lo que podemos proceder con la escalada de privilegios.

# Abuso de la capability en Python3.8
Para escalar privilegios, primero necesitamos setear el UID del contexto en el que estamos a 0. Para ello, deebmos ejecutar el binario e importar la librería **os*:

	python3.8
    >>> import os

Posteriormente, seteamos el UID a cero:

	>>> os.setuid(0)

Con el UID en cero, solo nos resta spawnear una **sh** por medio de una llamada a **system**:

	>>> os.system("sh")

Esto nos devolvera una bash como el usuario root.

![20]

Por último, solo nos resta obtener la flag de root para finalizar la máquina:

![21]
	
[0]:/assets/images/cap-htb/0.png
[1]:/assets/images/cap-htb/1.png
[2]:/assets/images/cap-htb/2.png
[3]:/assets/images/cap-htb/3.png
[4]:/assets/images/cap-htb/4.png
[5]:/assets/images/cap-htb/5.png
[6]:/assets/images/cap-htb/6.png
[7]:/assets/images/cap-htb/7.png
[8]:/assets/images/cap-htb/8.png
[9]:/assets/images/cap-htb/9.png
[10]:/assets/images/cap-htb/10.png
[11]:/assets/images/cap-htb/11.png
[12]:/assets/images/cap-htb/12.png
[13]:/assets/images/cap-htb/13.png
[14]:/assets/images/cap-htb/14.png
[15]:/assets/images/cap-htb/15.png
[16]:/assets/images/cap-htb/16.png
[17]:/assets/images/cap-htb/17.png
[18]:/assets/images/cap-htb/18.png
[19]:/assets/images/cap-htb/19.png
[20]:/assets/images/cap-htb/20.png
[21]:/assets/images/cap-htb/21.png