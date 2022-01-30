---
layout: single
title: Máquina Kenobi - TryHackMe (OSCP Style) (Offensive Pentesting Path)
excerpt: "Siguiendo con el path Offensive Pentesting de TryHackMe, hoy nos toca resolver la máquina Kenobi, la cual requiere de una buena enumeración y un poco de intuición."
date: 2022-01-30
classes: wide
header:
  teaser: /assets/images/kenobi/portada.png
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

# Información de la máquina
La máquina se encuentra en la plataforma [TryHackMe](https://tryhackme.com/room/kenobi) y es la última de la sección Getting Started del path Offensive Pentesting.

# Reconocimiento
La máquina tiene un sistema operativo Linux. Podemos corroborarlo con una traza **ICMP**, la cual nos devuelve un TTL de 64:

![1]

Por medio de un escaneo de puertos con nmap, detectamos aquellos que se encuentren activos con un estatus **open** en la máquina víctima:

	nmap --min-rate 5000 -p- --open -Pn -n 10.10.86.149	-oN portScan

![2]

Vemos que se encuentran habilitados varios puertosa entre los que destacan el 21 (FTP), el 22 (SSH), el 80 (HTTP), el 139 y 445 (SMB) y el 111 (RPC).

Si realizamos un escaneo más exhaustivo en busca de servicios y versiones específicas de dichos puertos, nos encontramos con la siguiente información:

![3]

Esto nos indica varias cosas que enumerare a continuación:

1. El software que controla el servicio FTP es un **ProFTPD versión 1.3.5**.
2. El servidor web tiene una entrada deshabilitada en el robots.txt, la cual es **admin.html**.
3. El servicio SMB está administrado por un Samba en Ubuntu.
4. El servicio RPC proporciona acceso a un NFS (Networw File System), es decir, una carpeta compartida en red.

# FTP
Iniciamos con el primer servicio el cual es un FTP. Intentamos loguearnos con una cuenta **anonymous**, sin embargo, el servidor nos impide el acceso.

![4]

Por el momento dejaremos el servicio FTP, pero posteriormente regresaremos a él.

# Servidor Web
La página web muestra una imágen estática de la película **Star Wars - Episodio III: La Venganza de los Sith** en la icónica escena de batalla entre Obi-Wan Kenobi y Anakin Skywalker.
	
![5]

Si nos dirigimos al directorio que aparecía oculto (**admin.html**), nos encontraremos con un GIF que muestra al Almirante Ackbar (Star Wars) diciendo su icónica frase **"It's a trap"**. Se tomó esto como un indicativo de que este no es el camino para comprometer la máquina, por lo que volvemos a la enumeración de los demás servicios.
	
![6]

# Servicio SMB
El siguiente servicio a enumerar es SMB. Primero, enumeramos todos los recursos compartidos con los que cuenta el servidor:

	smbclient -L 10.10.191.191 -N
	
La **-N** de al final del comando indica que queremos realizar el listado de recursos sin proporcionar credencial de autenticación alguna (dado que no la tenemos). Vemos que nos devuelve el siguiente resultado:

![7]

Una carpeta resalta de las demás, **anonymous**. Para conectarnos a ella, empleamos nuevamente **smbclient**:

	smbclient //10.10.191.19/anonymous -N
	
Nuevamente, la **-N** indica que no proporcionaremos ninguna credencial para ejecutar la consulta. Afortunadamente podemos acceder al recurso donde se encuentra un único archivo llamado **log.txt**.

![8]

Este archivo podemos traerlo a nuestra máquina con el comando **get**. Una vez lo tengamos descargado, accedemos para ver su contenido:

![9]

El archivo es bastante extenso, y contiene información sobre diversas configuraciones de servicios que vimos en nuestro escaneo inicial. Destaca sobre todo la parte inicial, donde vemos la creación de una llave privada para el usuario kenobi y su ubicación en el sistema.

Nuestro objetivo ahora es el de obtener dicha llave para poder conectarnos vía **SSH** y acceder a la máquina.

# Volvemos al FTP
En nuestro primer escaneo obtuvimos la versión del servicio FTP, la cual es un ProFTPD 1.3.5. Si buscamos posibles vulnerabilidades para esta versión nos encontraremos con lo siguiente:

![10]

Esta vulnerabilidad para ProFTPD se conoce como **"mod_copy"**, y nos permite realizar copias de archivos dentro del sistema de manera arbitraria sin la necesidad de estar debidamente autenticados. Esto se realiza por medio de los comandos **CPFR** (para copiar) y **CPTO** (para "pegar", visto de una manera). Esto lo podemos observar en el código fuente de los exploits.

![11]

Estos exploits buscan aprovecharse de dicha vulnerabilidad para poder ejecutar comandos de manera remota. Sin embargo, nosotros no necesitamos eso, por lo que solo será necesario ejecutar los comandos anteriores para obtener la llave privada del usuario **kenobi**.

El procedimiento está explicado a detalle en [este video](https://www.youtube.com/watch?v=CVjirCpI0cM)

Para corroborar que tengamos posibilidad de ejecutar estos comandos, primero listamos aquellos que el servicio reconozca con ayuda de:

	site help
	
![12]

Como podemos observar, podemos ejecutar tanto **CPFR** como **CPTO**. Así que primero copiamos la llave privada con **CPFR**:

	site CPFR /home/kenobi/.ssh/id_rsa
	
Y la "pegamos" en el directorio donde se ubica el servicio web (ya que podemos acceder a este por medio del navegador) con ayuda de **CPTO**:

	site CPTO /var/www/html/id_rsa
	
Sin embargo, ocurre lo siguiente:

![13]

Resulta que no tenemos permisos de escritura dentro de este directorio, por lo que no podemos pegar la copia ahí. ¿A qué otro directorio podríamos tener acceso desde afuera para poder acceder a la copia de la llave provada?

# NFS
Recordemos que el servidor corre un RPC en el puerto 111 que nos ayuda a acceder a un NFS en el sistema. Si tenemos acceso a esa carpeta compartida desde afuera, podríamos intentar copiar nuestra llave ahí. Para listar los **NFS** con los que cuenta el sistema, ejecutamos el siguiente comando:

	showmount -e 10.10.191.191
	
![14]

Vemos que la carpeta que se está compartiendo a nivel de red es **/var**. Por lo tanto, intentemos copiar la llave privada a este directorio, de tal manera que posteriormente podamos hacer una montura del **NFS** y acceder al fichero.

![15]

Vemos que la copia se realizó correctamente, ahora solo nos resta montar el **NFS**. Esto se realiza creando primero una carpeta en el directorio **/mnt** de nuestra máquina, realizamos la montura y por último nos dirigimos al directorio para ver su contenido:

	mkdir /mnt/kenobi
	mount 10.10.191.191:/var/tmp /mnt/kenobi
	cd /mnt/kenobi
	
Dentro nos encontraremos con la llave privada **id_rsa**.

![16]

# Acceso al sistema
Para poder emplear dicha llave privada para autenticarnos por **SSH**, primero debemos darle los permisos necesarios:

	chmod 600 id_rsa
	
Por último, nos conectamos por **SSH** empleando el usuario **kenobi** y la llave privada:

	ssh kenobi@10.10.191.191 -i id_rsa
	
![17]

Ya habiendo ganado acceso al sistema, listamos la flag de usuario.

![18]

#Escalada de privilegios
Para escalar privilegios, primero se intentó listar los binarios con permiso **SUDO**. Por desgracia, esta acción solicitaba la contraseña del usuario **kenobi** (la cual desconocemos).

Acto seguido nos fuimos a buscar ficheros con permisos **SUID**, obteniendo lo siguiente:

![19]

Hay un binario que no corresponde a uno típico de Linux, el cual es **menu**. Dicho binario no es nativo ni forma parte del listado de binarios en **GTFObins**. Cuando uno se encuentra con este tipo de binarios, seguramente se trata de un binario custom creado por un usuario de la máquina. Para corroborarlo, lo ejecutamos normalmente:

![20]

Vemos que es una utilidad que realiza diversas acciones dentro de la máquina y muestra información. Lo que está pasando por detrás, es que ejecuta otros comandos dentro de la máquina, lo cual puede ser peligroso si es que los binarios a los que manda a llamar no están declarados con su ruta absoluta.

Recordemos que un binario puede ser llamado de forma relativa:

	curl
	
O de manera absoluta:
	
	/usr/bin/curl
	
Si se llama de manera relativa, existe un riesgo de seguridad, ya que se puede aplicar un **PATH Hijacking** en el sistema, de tal forma que yo como atacante puedo forzar al sistema a que busque el binario **curl** en un directorio de mi elección, donde previamente he colocado mi propia versión de **curl** que ejecuta una acción maliciosa.

Para poder corroborar esto, podemos imprimir los **strings** del binario:

	strings /usr7bin/menu
	
![21]

Vemos que, efectivamente, el comando **curl** es llamado de manera relativa, por lo que podemos aplicar un **PATH hijacking**. Para esto primero creamos un binario **curl** malicioso. Este no puede ser creado en la máquina víctima, dado que no cuenta con **nano**, así que primero lo creamos en nuestra máquina de atacante y, posteriormente, lo enviaremos a la máquina víctima.

![22]

Como de costumbre, vamos a convertir a /bin/bash en **SUID** para acceder como root directamente en la máquina, pero recuerda que aquí puede ir el comando que quieras (como una reverse shell).

Ya transferido a la máquina, debemos darle permisos de ejecución:

	chmod +x curl
	
Y posteriormente, modificar el PATH. Recordemos que el PATH es la variable de entorno que contiene las ubicaciones de los binarios en Linux, y que el sistema buscará de manera descendente para ejecutarlos. Sabiendo esto, debemos colocar nuestro directorio actual de trabajo (**/tmp**) como el primero en el PATH. Para esto, ejecutamos el siguiente comando:

	export PATH=/tmp:$PATH
	
De esta manera le decimos que queremos que el directorio **/tmp** sea el primero en el PATH y, posteriormente, los demás que ya estaban presentes.

![23]

Si ahora ejecutamos el binario **menu**, y seleccionamos la opción 1 (la que manda llamar a **curl**), veremos que no recibimos resultado alguno. Sin embargo, si ahora listamos los permisos del binario **/bin/bash**, veremos que ahora es **SUID**.

![24]

Lo único que resta es ejecutar el bash en modo persistente para conservar los permisos **SUID**:

	bash -p
	
![25]

Y listo, con esto hemos comprometido la máquina y ya podemos ver la flag de root.

[1]:/assets/images/kenobi/1.png
[2]:/assets/images/kenobi/2.png
[3]:/assets/images/kenobi/3.png
[4]:/assets/images/kenobi/4.png
[5]:/assets/images/kenobi/5.png
[6]:/assets/images/kenobi/6.png
[7]:/assets/images/kenobi/7.png
[8]:/assets/images/kenobi/8.png
[9]:/assets/images/kenobi/9.png
[10]:/assets/images/kenobi/10.png
[11]:/assets/images/kenobi/11.png
[12]:/assets/images/kenobi/12.png
[13]:/assets/images/kenobi/13.png
[14]:/assets/images/kenobi/14.png
[15]:/assets/images/kenobi/15.png
[16]:/assets/images/kenobi/16.png
[17]:/assets/images/kenobi/17.png
[18]:/assets/images/kenobi/18.png
[19]:/assets/images/kenobi/19.png
[20]:/assets/images/kenobi/20.png
[21]:/assets/images/kenobi/21.png
[22]:/assets/images/kenobi/22.png
[23]:/assets/images/kenobi/23.png
[24]:/assets/images/kenobi/24.png
[25]:/assets/images/kenobi/25.png