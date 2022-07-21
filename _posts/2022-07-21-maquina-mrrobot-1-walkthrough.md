---
layout: single
title: Máquina Mr Robot CTF - TryHackMe (OSCP Style) (Offensive Pentesting Path)
excerpt: "Esta máquina forma parte del apartado final del path. Se trata de una máquina Linux que deberemos rootear con el objetivo de obtener en el proceso un total de tres flags. Sin embargo, tiene un problema a mi parecer que veremos al momento de obtener acceso."
date: 2022-07-21
classes: wide
header:
  teaser: /assets/images/mrrobot/portada.jpg
  teaser_home_page: true
categories:
  - Comunidad
  - Blog
  - Writeup
tags:
  - wordpress
  - xmlrpc
  - bruteforce
  - tty
  - nmap
  - suid
---

# Introducción
La máquina se encuentra en [TryHackMe](https://tryhackme.com/room/mrrobot). Es una máquina Linux temática de Mr. Robot, la serie de televisión, y de esto podremos percatarnos al momento de acceder al servicio HTTP, donde tendremos una terminal y videos de la serie.

En mi opinión, presenta un enorme problema para tratarse de una máquina Beginner y CTF, y es que uno de los pasos consiste en realizar un Brute Forcing a un login con un diccionario que nos proporcionan con más de 850,000 entradas, el cual tardaría horas en ser completado en un equipo básico como el mío, razón por la cual llegó un punto en el que no sabía si estaba yendo por el camino correcto.

Solo para este caso, tuve que verificar con otras personas estar llevando a cabo el paso correcto, y percatarme de que no hay otro modo de avanzar; así que, como no soy capaz de dejar la máquina en standby bruteforceando el login, tuve que obtener la contraseña de otros Writeups (si dejo mi máquina haciendo este procedimiento, por sus características terminaría freezando pasada media hora).

# Índice
- [Introducción](#introducción)
- [Índice](#índice)
- [Escaneo de puertos](#escaneo-de-puertos)
- [Servicio HTTP](#servicio-http)
  - [Análisis del Wordpress](#análisis-del-wordpress)
  - [Robots (Primera flag)](#robots-primera-flag)
  - [Fuerza bruta contra el XMLRPC](#fuerza-bruta-contra-el-xmlrpc)
  - [Acceso al Wordpress](#acceso-al-wordpress)
- [Reverse shell y acceso al sistema](#reverse-shell-y-acceso-al-sistema)
- [Escalada de privilegios](#escalada-de-privilegios)
  - [Contraseña guardada para **robot**](#contraseña-guardada-para-robot)
  - [Cambio de la bash](#cambio-de-la-bash)
  - [Volviendo al SUID nmap (segunda y tercera flag)](#volviendo-al-suid-nmap-segunda-y-tercera-flag)

Escaneo de puertos
==================================================================================================================
Verificamos que se trata de una máquina Linux por medio de una traza ICMP.

![1]

Iniciamos con un escaneo básico con **nmap** en busca de puertos abiertos en el sistema víctima:

    nmap -sS --min-rate 5000 -p- --open -Pn -n -vv 10.10.142.60 -oG allPorts

![2]

Observamos que hay dos puertos abiertos: el HTTP y el HTTPS. Realizamos un escaneo a profundidad de dichos puertos en busca de servicios y versiones que corren en ellos.

    nmap -sC -sV -p80,443 10.10.142.60 -oN targeted

![3]

Vemos que en este segundo escaneo, ahora muestra los puertos como filtered. Y es que si intentamos correr el nmap anterior, veremos que ya no encontrará puerto alguno. Sin embargo, vamos a analizarlos.

Servicio HTTP
==================================================================================================================
Probamos analizar los servicios con ayuda de whatweb, sin embargo, no es capaz de obtener información al respecto:

![4]

Si entramos por medio de un navegador al servicio web veremos una especie de terminal.

![5]

Esta terminal cuenta con un par de comandos personalizados, cada uno de ellos nos llevará a diferentes directorios donde podremos ver algunas imágenes, documentos y videos de la serie Mr Robot. Sin embargo, nada de esto es relevante para nuestro análisis ni acceso al sistema.

Analizando el código fuente de uno de estos directorios, me encuentro con un comentario donde indica que hay un tema instalado en el directorio de Wordpress, lo cual me lleva a creer que este servicio HTTP corre un Wordpress.

Nota: El servicio HTTPS es un espejo del HTTP, es decir, muestran exactamente la misma información.

## Análisis del Wordpress
Ingresamos al sitio de logueo para confirmar que se trata de un Wordpress:

    /wp-admin.php

![6]

Como bien sabemos, el panel de logueo de este CMS nos permite enumerar usuarios válidos en el sistema al ingresar nuestra suposición en el formulario y, de acuerdo a la salida que obtengamos, sabremos si el usuario existe. Intentamos *admin*, pero este no existe.

Recordando que se trata de una máquina temática, pruebo nombres de personajes de la serie, empezando por los evidentes:

    mrrobot
    elliot

Para mi suerte, el usuario *elliot* existe.

![7]

Como revisamos en una máquina anterior, el servicio XMLRPC nos permite realizar un ataque de fuerza bruta en busca de credenciales de Wordpress válidas. Accedemos al archivo para corroborar si tenemos acceso y volveremos a este más tarde:

![8]

## Robots (Primera flag)
Un escaneo realizado con ayuda de **WPScan** nos revela que el servicio web cuenta con un archivo *robots.txt*:

    wpscan --url http://10.10.142.60

![9]

Dentro de este archivo, veremos que hay dos ubicaciones ocultas:

    fsocity.dic
    key-1-of-3.txt

![10]

Uno de ellos es, evidentemente, la primera flag que nos solicita la plataforma:

![11]

Mientras que el segundo se trata de un diccionario de palabras con más de 850,000 entradas.

![12]

## Fuerza bruta contra el XMLRPC
Como ya lo mencioné, recordemos que al tener acceso al XMLRPC podemos llevar a cabo un ataque de fuerza bruta para poder obtener credenciales válidas del Wordpress. Para ello, corroboramos que podamos ejecutar la instrucción **wp.getUserBlogs** con ayuda de BurpSuite.

![13]

Evidentemente, esta ejecución nos devolverá que la contraseña o usuario son incorrectos, lo que nos indica que todo está funcionando correctamente. Ahora, aquí viene la parte fea y rara de la máquina en mi opinión.

Es evidente que este es el camino a seguir para lograr acceder al Wordpress y obtener una reverse shell, y más sabiendo que contamos con un diccionario que emplear para el ataque; sin embargo, al momento de iniciarlo con mi [script](https://github.com/blu3ming/XMLRPC-Brute-Force), me di cuenta de que tardaría un total de 7 días en completarse (XD), dado que no tiene bien implementados los hilos aún.

Por esta razón, intenté ahora emplear nuevamente herramientas de otros developers que encontré en GitHub, entre ellos, uno que permite declarar la cantidad de hilos que requerimos: [xmlrpc-bruteforcer](https://github.com/aress31/xmlrpc-bruteforcer). Aún con esto, no podemos declarar una enorme cantidad de hilos, dado que saturamos la red y muchos de los request no se harían correctamente, encontrándonos con falsos negativos.

Investigando más, me encuentro con un usuario que afirma que con la herramienta **WPScan** puedes declarar una cantidad de 10,000 hilos y que el programa terminaría el ataque en un total de 34 minutos, así que me decido a intentarlo. Por desgracia, aún con esa enorme cantidad de hilos, el ataque mostraba un tiempo estimado de 2 horas, mismas que iban aumentando cada que pasaba el tiempo; primero, no puedo dejar el ataque corriendo ahí en segundo plano dado que las máquinas en TryHackMe tienen un tiempo de actividad de dos horas, y cada cierto tiempo tenemos que estar extendiendo este tiempo manualmente; y segundo, mi computadora batalla al ejecutar la máquina virtual de Kali (por eso muchas veces ocupaba la Attacking Machine de TryHackMe), y si dejo el proceso corriendo, sin atender cada cierto tiempo a la máquina, esta se congela por completo sin dejarme controlarla nuevamente, obligándome a cerrar todo haciendo que el ataque no tenga sentido.

Es por esta razón que tuve que leer otros Writeups y encontrarme con la contraseña correcta, de otra manera no habría manera de que pudiera completar la máquina; acordemos que ya tenía la idea del camino y solo me detenía la potencia de mi computadora. El ataque fue exitoso, siendo las credenciales válidas:

    elliot:ER28-0652

Y lo que es peor, esta contraseña está en los últimos lugares del diccionario, obligándonos a recorrerlo todo en el ataque para encontrarla.

![14]

## Acceso al Wordpress
Como las credenciales son válidas, logramos acceder al panel de administración del CMS.

![15]

Ya sabemos qué hacer a partir de ahora, modificar la plantilla de la página 404 del tema instalado en el Wordpress, subiendo una reverse shell en php y guardando los cambios.

![16]

Reverse shell y acceso al sistema
==================================================================================================================
Ya con la reverse shell en el Wordpress, accedemos al directorio del 404 o a cualquier página que no exista en el servidor, ya con una consola en escucha con netcat. Vemos cómo obtenemos la shell en el sistema.

![17]

Sin embargo, al momento de realizar el tratamiento de la TTY, me encuentro con que no existe la terminal XTERM en este, complicándome el poder obtener una shell completamente interactiva:

![18]

Escalada de privilegios
==================================================================================================================
Dado que no tenemos una terminal interactiva, lo único que podemos hacer es spawnear una con un prompt decente:

    script /dev/null -c bash

Por ahora, esto nos servirá para trabajar (no tendremos autocompletado, limpieza ni CTRL+C). Primero, buscamos por permisos SUID, encontrándonos con que el binario de **nmap** cuenta con estos.

    find / -perm -u=s 2>/dev/null

Para poder spawnear una shell a partir de un nmap SUID, necesitamos que este cuente con el modo interactivo. Desde este modo, podemos indicarle que ejecute un comando, como darnos una shell como el usuario propietario del permiso (root).

    /usr/local/bin/nmap --interactive

Sin embargo, como podemos observar, al momento de querer entrar a este modo interactivo, la consola nos cierra toda posibilidad de ingresar datos.

![19]

## Contraseña guardada para **robot**
Siguiendo con la enumeración, vemos que en el directorio del usuario **robot** está la segunda flag, la cual tiene permisos de lectura bloqueados para cualquiera que no sea dicho usuario. Por otro lado, tenemos un archivo llamado *password.raw-md5*, al cual sí podemos acceder, encontrándonos con lo que parecen ser unas contraseñas hasheadas.

![20]

Pasándola por *crackstation*, vemos que la contraseña en texto plano es el alfabeto completo:

![21]

Ahora, si intentamos cambiar de usuario empleando estas credenciales, nuevamente tenemos el problema de que la consola nos cierra la posibilidad de introducir datos, imposibilitando el poder realizar cualquier acción.

![22]

## Cambio de la bash
Investigando sobre esto, me encuentro con que hay una manera secundaria de spawnear una bash semi-interactiva empleando Python (recordemos que antes empleamos el script /dev/null). Para spawnear esta, ejecutamos:

    python -c "import pty;pty.spawn('/bin/bash')"

Esto nos devolverá un prompt igual al que ya teníamos, pero con la ventaja de que en esta ocasión ya tenemos un poco de interactividad con la bash; tanto así, que ya tenemos la oportunidad de introducir valores en la consola.

## Volviendo al SUID nmap (segunda y tercera flag)
Como ya podemos interactuar más con la bash, me salto el paso de cambiar de usuario, enfocándome nuevamente en el nmap SUID. Tratamos de entrar al modo interactivo, notando cómo ahora ya podemos interactuar con este; ejecutamos lo siguiente:

    !bash -p

![23]

Como podemos observar, nos devuelve una bash como el usuario *root*. Por último, ya como este usuario privilegiado, buscamos las dos flags que nos faltan de golpe.

![24]

[1]:/assets/images/mrrobot/1.png
[2]:/assets/images/mrrobot/2.png
[3]:/assets/images/mrrobot/3.png
[4]:/assets/images/mrrobot/4.png
[5]:/assets/images/mrrobot/5.png
[6]:/assets/images/mrrobot/6.png
[7]:/assets/images/mrrobot/7.png
[8]:/assets/images/mrrobot/8.png
[9]:/assets/images/mrrobot/9.png
[10]:/assets/images/mrrobot/10.png
[11]:/assets/images/mrrobot/11.png
[12]:/assets/images/mrrobot/12.png
[13]:/assets/images/mrrobot/13.png
[14]:/assets/images/mrrobot/14.png
[15]:/assets/images/mrrobot/15.png
[16]:/assets/images/mrrobot/16.png
[17]:/assets/images/mrrobot/17.png
[18]:/assets/images/mrrobot/18.png
[19]:/assets/images/mrrobot/19.png
[20]:/assets/images/mrrobot/20.png
[21]:/assets/images/mrrobot/21.png
[22]:/assets/images/mrrobot/22.png
[23]:/assets/images/mrrobot/23.png
[24]:/assets/images/mrrobot/24.png