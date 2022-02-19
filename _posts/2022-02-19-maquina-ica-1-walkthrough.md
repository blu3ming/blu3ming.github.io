---
layout: single
title: Máquina ICA:1 - VulnHub (OSCP Style)
excerpt: "Para variar un poco, decidimos intentar una máquina bastante sencilla de la plataforma VulnHub, esto luego de que haya sido recomendada por el canal de YouTube I.T Security Labs."
date: 2022-02-19
classes: wide
header:
  teaser: /assets/images/ica-1/portada.png
  teaser_home_page: true
categories:
  - VulnHub
  - Blog
  - Writeup
tags:
  - qdPM
  - MySQL
  - Base64
  - Hydra
  - SSH
  - SUID
  - Path Hijacking
---

# Introducción
La máquina se encuentra en la plataforma [VulnHub](https://www.vulnhub.com/entry/ica-1,748/) y figura con una dificultad fácil. Se trata de una máquina temática en la que debemos descubrir el proyecto secreto en el que está trabajando ICA. Esta máquina se descarga y se tiene que correr desde VirtualBox, nada de desplegarla de una plataforma.

Fue durante la resolución de esta máquina que mi Kali murió, así que iniciaremos con capturas de pantalla del sistema anterior y posteriormente cambiarán por las de mi nuevo Kali.

# Despliegue
Primero iniciamos la máquina desde VirtualBox. Para esto, descargamos la máquina desde la plataforma y en VirtualBox seleccionamos Archivo -> Importar servicio virtualizado.

Esta ya está configurada para iniciar y tomar una IP por DHCP de manera automática.

![1]

# Reconocimiento
Por medio de un escaneo de puertos con nmap, detectamos aquellos que se encuentren activos con un estatus **open** en la máquina víctima:

	nmap -p- --open -T5 -Pn -n 192.168.0.28 -oG allPorts

![2]

Vemos que se encuentran habilitados los servicios de SSH, HTTP y MySQL.

Cabe mencionar que olvidé realizar el escaneo exhaustivo en busca de versiones con nmap, sin embargo, la máquina es bastante sencilla que no lo requerimos; pero es algo que no debe pasar.

# Servicio HTTP
Si entramos al sitio web veremos un panel de logueo de un software llamado **qdPM** versión 9.2.

![3]

Buscando por una vulnerabilidad, nos encontramos con que cuenta con un password exposure, es decir, filtración de contraseñas para la base de datos (recordemos que el puerto de MySQL está habilitado).

![4]

Su explotación es de lo más sencilla y se basa en simplemente acceder a una dirección específica del servidio web.

    http://192.168.0.28/core/config/databases.yml

Procedemos a descargarlo desde **curl**:

![5]

Este archivo muestra el usuario y contraseña para la base de datos (y sí, en texto claro).

![6]

# Enumeración MySQL
Para acceder a MySQL de manera remota podemos valernos de la herramienta del mismo nombre:

    mysql -u qdpmadmin -h 192.168.0.28 -p

Con esto le indicamos a la herramienta el usuario, el hosto remoto y que vamos a autenticarnos con una contraseña, la cual nos pedirá en cuanto ejecutemos el comando.

![7]

Como podemos ver, las credenciales son válidas y ahora podemos enumerar las bases de datos en busca de más credenciales para poder acceder al sistema. Para ello, primero listamos las bases de datos:

    show databases;

![8]

La base de datos de nombre **staff** parece tener información sobre el personal de la empresa, por lo que podemos suponer que contiene credenciales en su interior. Para trabajar con ella, empleamos la instrucción:

    use staff;

Posteriormente, podemos listar las tablas con las que cuenta la base de datos en busca de información de interés:

    show tables;

![9]

Vamos a enumerar el contenido tanto de la tabla **login** como de **user**. Para ello, solo debemos ejecutar la instrucción:

    select * from login;

![10]

Esta tabla contiene contraseñas y las identifica por un id de usuario a cada una. Sin embargo, no tenemos usernames. Para ello, enumeramos la tabla faltante:

    select * from user;

![11]

Esta tabla nos devuelve un listado de usuarios, y ya con estos, podemos crear un pequeño diccionario que incluya tanto el nombre como aparece en la base de datos, como el mismo nombre pero en minúsculas (recordemos que generalmente los nombres de usuario son solo en minúsculas, así que cubrimos esa posibilidad).

![12]

Por otro lado, creamos un diccionario con las contraseñas que encontramos:

![13]

Estas están codificadas en base64, y lo delata el símbolo de igual (=) al final de cada una. Para descifrarlas, lo pasamos por la siguiente instrucción en bash:

    while read p; do echo $p | base64 -d;echo; done<passwords

![14]

Esta instrucción nos devolverá las contraseñas ya descifradas, así que las guardamos en un diccionario que usaremos con el listado de usuarios en un ataque de fuerza bruta en busca de credenciales válidas.

# Fuerza bruta a SSH
Dado que solo nos resta un servicio por analizar en la máquina, probamos el ataque de fuerza bruta contra SSH en busca de credenciales válidas. Para ello, empleamos la herramienta Hydra.

    hydra -L usuarios -P b64_passwords ssh://192.168.0.28 -f

Con esto le indicamos a Hydra que emplee como usuario de entrada cada renglón del archivo usuarios, lo mismo para las contraseñas y que ataque el servicio SSH del servidor remoto.

![15]

Como podemos ver, el ataque es exitoso y nos devuelve credenciales válidas para el usuario **travis**. Recordemos que la herramienta se detiene cuando encuentra credenciales válidas, por lo que siempre vale la pena intentar nuevamente con las entradas restantes (como buena práctica, y solo cuando sea posible editar el diccionario; es decir, que no sea demasiado grande). En este caso, solo dejamos los usuarios restantes después de **travis** y volvemos a correr la herramienta; veremos que podemos obtener otro usuario válido:

![16]

![17]

Primero intentemos acceder a la máquina con las credenciales del usuario **travis**.

# SSH
Vemos que las credenciales obtenidas son correctas, logrando acceder a la máquina remota con una consola estable como solo SSH lo puede ofrecer. Listamos la flag de usuario para evidenciar el acceso:

![18]

# Escalada de privilegios
Es hora de enumerar el sistema en busca de posibles vectores de ataque para escalar privilegios. Primero, intentamos listar posibles permisos sudo que tenga nuestro usuario:

![19]

Vemos que **travis** no tiene permitido la ejecución de **sudo** en el sistema, por lo que proseguimos. Listamos ahora binarios con permisos SUID:

    find / -perm -u=s 2>/dev/null

![20]

Dentro del listado veremos un binario que no es típico de un sistema Linux, el cual se llama **get_access**. Si lo ejecutamos veremos la siguiente salida:

![21]

Al parecer lista un par de datos sobre la máquina. Como vimos en máquinas anteriores, si un binario es llamado de manera relativa y no absoluta, puede generar una vulnerabilidad de la que nos podemos aprovechar por medio de un Path Hijacking; sobre todo cuando se trata de un binario SUID, ya que al ser ejecutado, realizará la acción que le indiquemos como el usuario privilegiado. Para corroborar esto, mostramos las strings del binario:

    strings /opt/get_access

![22]

Como podemos observar, la herramienta ejecuta aparte el binario **cat**, y este es llamado de manera relativa sin la ruta absoluta; el candidato perfecto para un Path Hijacking. Este proceso ya lo hemos explicado con anterioridad, así que solo seguimos el procedimiento. Primero creamos nuestro propio **cat** que llevará a cabo, adivinaste, un cambio de permisos SUID a la bash (prometo cambiar esto para la siguiente máquina para intentar nuevos enfoques).

![23]

Por último, solo modificamos el PATH para que incluya nuestro directorio con el **cat** "malicioso".

![24]

Y listo, solo necesitamos ejecutar el binario SUID. Veremos que ya no muestra el contenido que vimos con anterioridad, por lo que podemos suponer que la herramienta se ejecutó como queríamos. Lo comprobamos con un vistazo a los permisos de la bash, viendo que ahora son SUID.

![25]

Solo nos resta ejecutar la bash con la bandera -p para obtener una consola como el usuario administrador, habiendo comprometido la máquina correctamente.

Cabe destacar que si intentamos ver la flag de root, debemos recordar que ahora **cat** no funciona, ya que siempre estará llamando a nuestro **cat** personalizado. Para poder emplear el cat de siempre, deberemos llamarlo de manera absoluta:

    /bin/cat root.txt

![26]


[1]:/assets/images/ica-1/1.png
[2]:/assets/images/ica-1/2.png
[3]:/assets/images/ica-1/3.png
[4]:/assets/images/ica-1/4.png
[5]:/assets/images/ica-1/5.png
[6]:/assets/images/ica-1/6.png
[7]:/assets/images/ica-1/7.png
[8]:/assets/images/ica-1/8.png
[9]:/assets/images/ica-1/9.png
[10]:/assets/images/ica-1/10.png
[11]:/assets/images/ica-1/11.png
[12]:/assets/images/ica-1/12.png
[13]:/assets/images/ica-1/13.png
[14]:/assets/images/ica-1/14.png
[15]:/assets/images/ica-1/15.png
[16]:/assets/images/ica-1/16.png
[17]:/assets/images/ica-1/17.png
[18]:/assets/images/ica-1/18.png
[19]:/assets/images/ica-1/19.png
[20]:/assets/images/ica-1/20.png
[21]:/assets/images/ica-1/21.png
[22]:/assets/images/ica-1/22.png
[23]:/assets/images/ica-1/23.png
[24]:/assets/images/ica-1/24.png
[25]:/assets/images/ica-1/25.png
[26]:/assets/images/ica-1/26.png