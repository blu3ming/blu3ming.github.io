---
layout: single
title: Máquina Blue - TryHackMe (Offensive Pentesting Path) - EternalBlue sin Metasploit
excerpt: "Siguiendo con el path Offensive Pentesting de TryHackMe, ahora nos toca comprometer la máquina Blue. Esto se realizará sin ayuda de Metasploit ejecutando el exploit EternalBlue de manera <<manual>>."
date: 2022-01-29
classes: wide
header:
  teaser: /assets/images/blue/portada.gif
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
La máquina se encuentra en la plataforma [TryHackMe](https://tryhackme.com/room/blue). Contrario a lo que nos indica la plataforma, esta guía será un **boot2root** con el objetivo de demostrar cómo explotar una vulnerabilidad **EternalBlue** sin ayuda de Metasploit. Eso quiere decir que no llevaremos a cabo las acciones post-explotación que solicita la máquina. El único propósito del sistema es el de fungir como playground para probar dicho exploit. Si buscas completar el room en TryHackMe, te recomiendo mejor seguir con el procedimiento que te indica la plataforma (con Metasploit).

¿Por qué? Durante la explotación del EternalBlue me encontré con varios fallos y errores que logré corregir, por lo que esta guía buscará además mostrar posibles fallos de ejecución que se pudieran presentar a la hora de explotar esta vulnerabilidad y cómo arreglarlos para continuar. En una entrada posterior intentaremos realizar la post-explotación que propone TryHackMe.

# Reconocimiento
La máquina es un Windows 7 de 64 bits. Nos lo indica la misma página de TryHackMe y podemos corroborarlo con una traza **ICMP**:

![1]

Por medio de un escaneo de puertos con nmap, detectamos aquellos que se encuentren activos con un estatus **open** en la máquina víctima:

	nmap --min-rate 5000 -p- --open -Pn -n 10.10.153.51 -oN portScan

![2]

Vemos que se encuentran habilitados varios servicios de Windows como el SMB y varios puertos típicos de Windows.

Si realizamos un escaneo más exhaustivo en busca de servicios y versiones específicas de dichos puertos, nos encontramos con la siguiente información:

![3]

Siempre que nos enfrentamos a un sistema Windows 7, debemos corroborar si es que existe la posibilidad de poder explotar la vulnerabilidad **EternalBlue**. Para ello, podemos hacer uso de scripts **vuln** y **safe** en un escaneo con nmap enfocado al puerto **SMB** (445).

	nmap --script "vuln and safe" -p445 10.10.153.51 -oN vulnScan

![4]

Vemos que es vulnerable al **MS17-010**, más conocido como **EternalBlue**, por lo que procedemos a explotar esta vulnerabilidad.

# EternalBlue
Para explotar esta vulnerabilidad, podemos emplear este [repositorio](https://github.com/lokendrasinghrawat/AutoBlue-MS17-010). Lo descargamos por medio de un **git clone**.

![5]

Este exploit tiene dos fases para ejecutarlo:

1. Crear los shellcodes necesarios para Windows.
2. Ejecutar el exploit con dicho shellcode.

# Creación de los shellcodes
Aquí es donde inicia el proceso de explotación y donde podríamos encontrar varios errores. El script requiere de un shellcode que pueda ser empleado tanto para sistemas de 32 bits como de 64 bits, y para ello, el repositorio incluye una herramienta para crearlos automáticamente llamado **shell_prep.sh**. Este nos permitirá, además, añadir al shellcode nuestro reverse shell con el cual ganaremos acceso al sistema.

Para ejecutarlo, debemos ir al directorio **shellcode** y ejecutar el binario **shell_prep.sh**

	./shell_prep.sh
	
![6]

**Nota:** Como podemos observar en la imágen anterior, podríamos encontrarnos con un error que dice **"line 13: nasm: command not found"**. Si este es el caso, lo mejor es finalizar el programa con CTRL+C ya que no tiene sentido continuar. Para arreglarlo, solo debemos instalar el comando faltante:

	apt install nasm
	
![7]

Una vez instalado, volvemos a ejecutar el **shell_prep.sh** y veremos que dicho error ya no aparece; si este es el caso, podemos continuar.

![8]

Para crear los shellcodes con una reverse shell que podamos recibir con ayuda de netcat, deberemos dar los siguientes parámetros:

	- would you like to auto generate a reverse shell with msfvenom (Y/n)
	Y
	- LHOST for reverse connection:
	TU IP DONDE RECIBIRÁS LA REVERSE SHELL
	- LPORT you want x64 to listen on:
	PUERTO DONDE ESTARÁS A LA ESCUCHA
	- LPORT you want x86 to listen on:
	PUERTO DONDE ESTARAS A LA ESCUCHA (DADO QUE EL SISTEMA ES UN WINDOWS DE 64 BITS, ESTA OPCIÓN NO ES NECESARIA; SIN EMBARGO, NO PUEDE SER EL MISMO PUERTO QUE EL ANTERIOR)
	- Type 0 to generate a meterpreter shell or 1 to generate a regular cmd shell
	1 (COMO ESTA GUÍA SE BASA EN NO USAR METASPLOIT, NO QUEREMOS RECIBIR UNA REVERSE SHELL CON AYUDA DE METERPRETER)
	- Type 0 to generate a staged payload or 1 to generate a stageless payload:
	1 (STAGED ES PARA UNA REVERSE SHELL RECIBIDA CON METERPRETER; PARA RECIBIRLA CON NETCAT, DEBEMOS SELECCIONAR LA STAGELESS PAYLOAD).
	
![9]

Al finalizar la generación de los shellcodes, los unirá en uno solo un script llamado **eternalblue_sc_merge.py**, sin embargo, puede dar un error al finalizar que dice: **TypeError: must be str, not bytes**. Esto se debe a que el exploit está diseñado para ser usado con Python versión 2; si nuestro sistema Linux está configurado para emplear Python versión 3 por defecto al escribir en la línea de comandos **python**, entonces nos encontraremos con este error.

![10]

Para corregirlo, debemos modificar el script **shell_prep.sh**, específicamente en la línea 96; tenemos que cambiar el comando **python** por **python2** para que al ser ejecutado, este tire por Python versión 2.

![11]

Con esta corrección, volvemos a ejecutar el script con los mismos parámetros de antes. Veremos que al finalizar no nos lanza ningún error.

![12]

Ya tenemos todo listo, ahora solo nos resta ejecutar el exploit contra la máquina. Para ello, nos dirigimos al directorio principal del repositorio y ejecutamos el siguiente comando:

	python2 eternalblue_exploit7.py 10.10.113.175 shellcode/sc_all.bin
	
Debemos cuidar de colocar **python2** para que emplee dicha versión del lenguaje, además de ver que el script seleccionado sea el que tiene una terminación **7** (ya que se trata del script creado para los sistemas Windows 7). El último parámetro es la ubicación del shellcode creado anteriormente.

![13]

Como podemos ver, nos podemos llegar a encontrar con otro error (al menos el último con el que me pude topar), diciendo que no tenemos instalada la biblioteca **impacket**. Esto es normal, dado que esta biblioteca está instalada por defecto en la versión 3 de Python, pero no en la 2. Simplemente deberemos instalarla con el siguiente comando:

	python2 -m pip install impacket
	
![14]

Ahora sí, ya con la biblioteca instalada, podemos ejecutar nuevamente el script (sin olvidar ponernos en escucha con netcat en el puerto que hayamos especificado al momento de crear los shellcodes).

Obtendremos la siguiente salida:

![15]

# Acceso al sistema
Al haber ejecutado el script, veremos que recibimos una reverse shell desde netcat; de esta forma, ya tenemos acceso al sistema y no solo eso, sino que el acceso se ha ganado como un usuario privilegiado **NT AUTHORITY/SYSTEM**.

![16]

[1]:/assets/images/blue/1.png
[2]:/assets/images/blue/2.png
[3]:/assets/images/blue/3.png
[4]:/assets/images/blue/4.png
[5]:/assets/images/blue/5.png
[6]:/assets/images/blue/6.png
[7]:/assets/images/blue/7.png
[8]:/assets/images/blue/8.png
[9]:/assets/images/blue/9.png
[10]:/assets/images/blue/10.png
[11]:/assets/images/blue/11.png
[12]:/assets/images/blue/12.png
[13]:/assets/images/blue/13.png
[14]:/assets/images/blue/14.png
[15]:/assets/images/blue/15.png
[16]:/assets/images/blue/16.png