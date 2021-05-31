---
layout: single
title: Kioptrix Level 1 (Vulnhub) - WriteUp (by blu3ming)
excerpt: "Writeup de la máquina Kioptrix Nivel 1 de la plataforma Vunlhub. Nota: Puede incluir fallos y rabbit holes, los cuales se especifican, con el objetivo de que el lector no cometa los mismos errores que yo. Se recomienda leer el artículo completo antes de seguirlo al pie de la letra."
date: 2021-05-31
classes: wide
header:
  teaser: /assets/images/kioptrix-1/portada.png
  teaser_home_page: true
categories:
  - Writeup
tags:
  - Vulnhub
  - Writeup
  - Guia
---

[https://www.vulnhub.com/entry/kioptrix-level-1-1,22/](https://www.vulnhub.com/entry/kioptrix-level-1-1,22/)

+ Con una traza ICMP determinamos que en el servidor corre un sistema operativo indeterminado (TTL = 255). La máquina es un RedHat de Linux, por lo que se buscará una repetición de este valor en un futuro para determinar si se trata de un valor que se repite.

	![1]
	
+ Si realizamos un escaneo con nmap, veremos una serie de puertos y servicios que corren en el servidor. Entre ellos un SMB, HTTP, HTTPS, y SSH, entre otros.
  
    ``nmap -sS --min-rate 5000 -p- --open -vvv -n -Pn 10.10.10.3 -oG allPorts``
	
	![2]

+ Si realizamos un escaneo con Whatweb, solamente podremos detectar el servicio HTTP, no el HTTPS. En dicho servidor solo se está ejecutando un servidor Apache con una página por defecto para RedHat Linux.

    ![3]
    
+ Visitamos el servicio web desde un navegador, y corroboramos que únicamente hay una página por defecto de Apache.

    ![4]
	
	Ocurre el mismo escenario para el servicio HTTPS, el cual corre una versión de SSL muy antigua (de acuerdo con el navegador, la 1.1), por lo que se tuvo que cambiar a dicha versión en el nagevador para navegar por la página.
	
	![5]
    
+ Esto hizo que me cuestionara si, al ser un servicio bastante antiguo, no cuenta con alguna vulnerabilidad. Se realizó un escaneo de servicios y vulnerabilidades básico con nmap, el cual arrojó que el sistema corre un RedHat con Apache 1.3.20 y SSL 2.8.4.

    ![6]
    
+ Si realizamos una búsqueda en Google de estos servicios, con dichas versiones en específico, veremos que hay una vulnerabilidad de Remote Buffer Overflow, la cual sobrecarga el buffer y nos permite spawnear una shell como usuario root, realizando en un paso el acceso al sistema y la escalada de privilegios.
    
    ![7]
    
	El script requiere de que le especifiquemos el sistema operativo que corre el servidor, por lo que las únicas opciones que tenemos son **0x6a** y **0x6b**.

    ![8]
    
	Se ejecutó primero el exploit con la opción **0x6a**, pero no funcionó. Cuando se intentó con la segunda opción (**0x6b**), esta sí spawneó una shell como root. El script requiere la opción del sistema operativo, la IP del servidor, el puerto, y el número de conexiones a intentar para conectarse (por default, en el GitHub menciona que pueden ser 40, por lo que se probó ese número)

    ![9]
    
+ Se corrobora la escalada de privilegios en el sistema empleando para ello el comando whoami.
    
    ![10]
    
+ **Nota:** Al ser esta una máquina boot2root, no requerimos de encontrar ni reportar alguna flag del sistema, solamente escalar hasta ser usuario root de la máquina.
    
[1]:/assets/images/kioptrix-1/1.png
[2]:/assets/images/kioptrix-1/2.png
[3]:/assets/images/kioptrix-1/3.png
[4]:/assets/images/kioptrix-1/4.png
[5]:/assets/images/kioptrix-1/5.png
[6]:/assets/images/kioptrix-1/6.png
[7]:/assets/images/kioptrix-1/7.png
[8]:/assets/images/kioptrix-1/8.png
[9]:/assets/images/kioptrix-1/9.png
[10]:/assets/images/kioptrix-1/10.png