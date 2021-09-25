---
layout: single
title: Máquina Bastion - HackTheBox (OSCP Style)
excerpt: "Resolución de la máquina Bastion la cual figura en la plataforma con un nivel de dificultad Easy"
date: 2021-09-25
classes: wide
header:
  teaser: /assets/images/bastion/portada.png
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

La máquina figura en Hack The Box como una máquina Windows. Su dificultad es Easy, de 20 puntos.

![0]

# Reconocimiento
Iniciamos con una traza ICMP para detectar el Sistema Operativo que corre en el sistema. Por el TTL deducimos que es una máquina Windows (TTL aprox. de 128).

![1]

Por medio de un escaneo de puertos con nmap, detectamos aquellos que se encuentren activos con un estatus **open** en la máquina víctima:

![2]

Vemos que se encuentran habilitados una cantidad considerable de puertos. Por el momento, solo nos interesan unos cuantos: 22 (SSH), 139 Y 445 (SMB) y 5985 (WINRM).

Si realizamos un escaneo más exhaustivo en busca de servicios y versiones específicas de dichos puertos, nos encontramos con una gran cantidad de información:

	nmap -sC -sV -p22,135,139,445,5985,47001,4964,4965,4966,4967,4968,4969,4970 10.10.10.134 -oN targeted
    
![3]

A pesar de ser mucha la información reportada, no vemos nada que sea de relevancia por ahora o que nos indique algo que por el momento desconozcamos. Procedemos a hacer un reconocimiento de cada uno de los puertos reportados.

# Servicio SMB

Con ayuda de **crackmapexec** escaneamos este servicio en busca de información sobre el sistema víctima. Vemos que la máquina se llama **BASTION** y se trata de un Windows Server 2016 de 64 bits (aunque recordemos que esta información no siempre es la correcta, y usualmente, cuando crackmapexec reporta este Sistema Operativo, resulta ser al final un Server 2019). Por el momento quedémonos con la información que nos reporta.

![4]

Sin embargo no es la única enumeración al servicio SMB que se debe realizar. Ejecutamos **smbmap** en busca de recursos compartidos a nivel de red a los que podamos tener acceso. Para ello, y como no tenemos credenciales de momento, lo intentamos con un null session.

	smbmap -H 10.10.10.134 -u 'null'

![5]

Vemos una carpeta compartida llamada **Backups**, en la cual tenemos acceso de lectura/escritura. Si accedemos a este recurso por medio de **smbclient**, veremos el siguiente contenido:

	smbclient //10.10.10.134/Backups -N

![6]

Dado que la información dentro de este recurso es muy extensa, lo mejor será crear una montura de este sistema de archivos en nuestra máquina.

	mount -t cifs //10.10.10.134/Backups /mnt/smb

![7]

![8]

Hay un archivo en la raíz llamado **note.txt**, en el que alguna persona insta a los administradores a no transferir los *backups* de manera local, dado que la VPN de la oficina es demasiado lenta. Esto nos da a entender que en este sistema de archivos quizá exista algún backup del que obtener información.

![9]

Si seguimos analizando los directorios, nos encontraremos con unos archivos **VHD**. Este tipo de archivos son de **Disco Duro Virtual**, un tipo de archivo que almacena un sistema de ficheros completo de un Sistema Operativo que emplean programas como **Virtual Box** para cargar máquinas virtuales (aunque no es el único formato que acepta). Literalmente son copias completas e íntegras de un disco duro físico. Podemos corroborarlo por el tamaño que tiene este archivo (casi 5 GB).

![10]

# Montura de archivo VHD
Para poder montar este tipo de archivo y analizar su contenido, necesitaremos emplear la herramienta **qemu-nbd**. Si no la tenemos instalada, podemos hacerlo con el siguiente comando:

	sudo apt install qemu-utils
    
![11]

Posteriormente, para habilitar las unidades **nbd** que necesitaremos para montar el archivo **VHD**, ejecutaremos el siguiente comando:

	modprobe nbd
    
![12]

Podemos ver si las unidades fueron habilitadas si listamos el contenido del directorio **/dev**. La unidad que emplearemos para montar el disco duro virtual será la **nbd0**.

![13]

El proceso de montaje consite en dos pasos: primero debemos montar el **VHD** en la unidad **nbd0**:

	qemu-nbd -r -c /dev/nbd0 "/mnt/smb/WindowsImageBackup/L4mpje-PC/Backup 2019-02-22 124351/9b9cfbc4-369e-11e9-a17c-806e6f6e6963.vhd"
    
![14]

Este último comando lo que hace es tomar el archivo **VHD** y montarlo en una unidad nueva que toma como referencia a **nbd0**. Esta nueva unidad es **nbd0p1** la cual podemos ver si listamos de nuevo el contenido del directorio **/dev**.

El último paso consiste en montar esta nueva unidad en una carpeta del sistema. Para ello, creamos primero una carpeta en el directorio **/mnt** y realizamos el montaje con el comando **mount**.

	mount /dev/nbd0p1 /mnt/nbd
    
![15]

Este comando puede dar errores como el que se ve en la imagen. Sin embargo, no son importantes y no interfieren con la creación de la montura. Si listamos la carpeta donde esta fue creada, podemos ver el sistema de archivos correctamente.

![16]

# Dumpeo de hashes de usuario
Comenzamos a enumerar este sistema de archivos en busca de información relevante. Vemos si de casualidad existe la flag de usuario, pero esta no se encuentra. No hay archivos de relevancia en las carpetas de usuario. Lo que resta por hacer, es dumpear la SAM y obtener los hashes de usuario del sistema. Para ello, podemos dirigirnos a la ruta ``C:\Windows\System32\config``, dentro de este directorio encontraremos los archivos **SAM** y **SYSTEM**. Para dumpear los hashes, emplearemos *impacket-secretsdump*.

![17]

	impacket-secretsdump -sam SAM -system SYSTEM LOCAL
    
![18]

Este dumpeo nos devuelve hashes de usuario que podemos, o bien crackear por fuerza bruta, o bien usar para hacer **PassTheHash** y conectarnos al sistema. Intentamos primero este último escenario, probando el hash con crackmapexec tanto en los servicios **SMB** y **WINRM**.

![19]

Como podemos ver, el hash del usuario **L4mpje** es válido para el servicio **smb**, sin embargo, al no devolvernos un **Pwned!**, no podremos usar el hash para *PassTheHash*. El siguiente paso será intentar entonces crackear los hashes para obtener las contraseñas en texto claro.

![20]

Vemos que logra crackear solo la contraseña del **L4mpje**, la cual es ``bureaulampje``.

Dado que el sistema cuenta con el puerto 22 abierto (SSH), probamos a conectarnos a este servicio por medio de estas credenciales.

![21]

Vemos que son válidas, y ya tenemos acceso al sistema víctima.

# Acceso al sistema y escalada de privilegios
Ya con acceso al sistema, listamos la flag de usuario como primer paso.

![22]

Comenzamos a enumerar el sistema en busca de formas de escalar privilegios. Ejecutamos un ``whoami /all`` para ver con qué privilegios contamos en el sistema, y también los grupos a los que pertenecemos.

![23]

Este comando no nos reporta algo de utilidad, ya que los permisos no nos permiten ejecutar exploits conocidos como *JuicyPotato*, y los grupos no son de relevancia para escalar privilegios. Con esto, procedemos a enumerar los directorios del sistema. Empezamos con la raíz del sistema, es decir, la unidad *C:*. La primera carpeta es **Backups**; si la listamos, veremos el mismo contenido del recurso compartido homónimo.

![24]

La segunda carpeta se llama **Logs**, la cual contiene una enorme cantidad de archivos **evtx**, un tipo de archivo usado por Windows para hacer logs de eventos en el sistema. Lo usaría como último recurso, dado que son demasiados archivos como para revisar uno por uno con la esperanza de encontrar algo. Así que la dejamos en standby por el momento.

![25]

El siguiente directorio es *Program Files*, lugar donde se almacenan las aplicaciones que se instalan en el sistema. No vemos ningúna aplicación que sea relevante.

![26]

El siguiente directorio es *Program Files (x86)*, que tiene la misma función que el directorio anterior, solo que es para aplicaciones de 32 bits. Sin embargo, aquí vemos algo de interés: la carpeta de instalación de una aplicación llamada **mremoteNG**, la cual no es propia del sistema.

![27]

Esta aplicación cuenta con una forma de explotarlo con el objetivo de escalar privilegios en el sistema. Este procedimiento está explicado en la siguiente [página web](https://ethicalhackingguru.com/how-to-exploit-remote-connection-managers). Así que seguiremos los pasos descritos.

# Explotación de mRemoteNG
Básicamente, este ataque consiste en listar contraseñas hardcodeadas en un archivo de configuración de la aplicación, descifrarla con un script, y usarla para conectarnos al sistema como el usuario *Administrator*.

Primero, debemos buscar el archivo *confCons.xml*, el cual se encuentra en la ruta: ``C:\Users\L4mpje\AppData\Roaming\mRemoteNG``.

![28]

Si abrimos el archivo, veremos que almacena tanto usuarios como contraseñas de manera hardcodeada. Sin embargo, estas contraseñas no están en texto claro, hay que descifrarlas. Veremos las contraseñas del usuario *Asministrator* y *L4mpje*. Solo nos interesa la del usuario *Administrator*, ya que para el otro usuario ya tenemos credenciales de acceso.

![29]

Para descifrarlo, necesitaremos el siguiente repositorio llamado [mRemoteNG-Decrypt](https://github.com/haseebT/mRemoteNG-Decrypt). Simplemente le pasamos la contraseña que hemos encontrado por medio de la bandera **-s**.

![30]

# Acceso como NT AUTHORITY\SYSTEM
Esta herramienta nos devolverá la contraseña en texto claro, la cual ahora podemos probar por medio de **crackmapexec**. El resultado que nos da, es que para el servicio **WINRM** es válida y, sobre todo, vulnerable para **evil-winrm** (por el Pwned!).

	crackmapexec winrm 10.10.10.134 -u "Administrator" -p "thXLHM96BeKL0ER2"

![31]

Si nos conectamos por medio de *evil-winrm*, veremos que las credenciales son correctas y ahora hemos ganado acceso al sistema como el usuario administrador.

	evil-winrm -i 10.10.10.134 -u "Administrator" -p "thXLHM96BeKL0ER2"

![32]

Por último, solo listamos la flag de root. Con esto hemos finalizado de comprometer la máquina.

![33]
	
[0]:/assets/images/bastion/0.png
[1]:/assets/images/bastion/1.png
[2]:/assets/images/bastion/2.png
[3]:/assets/images/bastion/3.png
[4]:/assets/images/bastion/4.png
[5]:/assets/images/bastion/5.png
[6]:/assets/images/bastion/6.png
[7]:/assets/images/bastion/7.png
[8]:/assets/images/bastion/8.png
[9]:/assets/images/bastion/9.png
[10]:/assets/images/bastion/10.png
[11]:/assets/images/bastion/11.png
[12]:/assets/images/bastion/12.png
[13]:/assets/images/bastion/13.png
[14]:/assets/images/bastion/14.png
[15]:/assets/images/bastion/15.png
[16]:/assets/images/bastion/16.png
[17]:/assets/images/bastion/17.png
[18]:/assets/images/bastion/18.png
[19]:/assets/images/bastion/19.png
[20]:/assets/images/bastion/20.png
[21]:/assets/images/bastion/21.png
[22]:/assets/images/bastion/22.png
[23]:/assets/images/bastion/23.png
[24]:/assets/images/bastion/24.png
[25]:/assets/images/bastion/25.png
[26]:/assets/images/bastion/26.png
[27]:/assets/images/bastion/27.png
[28]:/assets/images/bastion/28.png
[29]:/assets/images/bastion/29.png
[30]:/assets/images/bastion/30.png
[31]:/assets/images/bastion/31.png
[32]:/assets/images/bastion/32.png
[33]:/assets/images/bastion/33.png