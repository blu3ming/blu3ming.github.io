---
layout: single
title: Máquina Inoculation - ReadySetExploit (OSCP Style)
excerpt: "Nuevamente nos moveremos de TryHackMe para intentar comprometer máquinas hechas por la comunidad. En este caso, la máquina Inoculation de ReadySetExploit nos permitirá aprender otro modo de explotar una SQLi."
date: 2022-02-22
classes: wide
header:
  teaser: /assets/images/inoculation/portada.jpg
  teaser_home_page: true
categories:
  - Comunidad
  - Blog
  - Writeup
tags:
  - SQLi
  - Burpsuite
  - Python library hijacking
---

# Introducción
La máquina se encuentra en el blog de [ReadySetExploit](https://readysetexploit.wordpress.com/) y se puede descargar desde [Google Drive](https://drive.google.com/file/d/1GvT38JMcoBfSnLk76TcDRQfh53mQUKS4/view?usp=sharing). Una vez descargada debemos agregarla a VirtualBox y cambiar la configuración de red a Bridged. Tardará un poco en arrancar y en obtener una IP por DHCP, así que verifica tus dispositivos de red para encontrar la IP correcta.

Esta máquina tiene dos enfoques: enseñar SQLi y Python Path Hijacking. El SQL injection es diferente a lo que hemos visto hasta ahora, ya que en lugar de introducir nuestras consultas por medio de la URL, ahora deberemos hacerlo por medio de los datos enviados por POST, y para ello nos auxiliaremos de Burpsuite.

# Despliegue
Primero iniciamos la máquina desde VirtualBox. Para esto, descargamos la máquina desde la plataforma y en VirtualBox seleccionamos Archivo -> Importar servicio virtualizado.

Deberemos cambiar su configuración de red a Bridged para que obtenga una IP.

# Reconocimiento
Por medio de un escaneo de puertos con nmap, detectamos aquellos que se encuentren activos con un estatus **open** en la máquina víctima:

	nmap -p- --open -T5 -Pn -n 192.168.0.30 -oG allPorts

![1]

Este proceso es muy lento, y si probamos un escaneo más rápido (con TCP SYN) los resultados son incorrectos. De hecho, como podemos ver en la imagen, ni siquiera nos reportó los puertos al finalizar, por lo que debemos agregar un verbose al comando para verlos a la par que se van descubriendo. Se descubrieron dos puertos, el 22 y el 8000.

Si realizamos un escaneo exhaustivo en busca de más información sobre estos puertos, la consulta no nos devolverá mayor información relevante:

    nmap -sC -sV -p22,8000 192.168.0.30 -oN targeted

![2]

# Servicio HTTP
Si entramos al sitio web veremos un panel de logueo:

![3]

Dado que no contamos con credenciales de acceso, intentamos primero un bypass con una inyección clásica:

    ' or 1=1-- -

Para fortuna nuestra, esto funciona, por lo que concluimos que el sitio es vulnerable a SQL injection. Al acceder al sitio, no vemos información relevante por ningún lado; es decir, logramos autenticarnos, pero no contamos con ninguna credencial que podamos usar para conectarnos ahora por SSH. Es por ello que deberemos ahora dumpear la base de datos en busca de credenciales válidas.

# SQL injection
Ahora ¿cómo podemos realizar esta SQLi si no parece haber entrada de datos donde introducir la query? Evidentemente, el panel de logueo es el campo donde debemos introducir nuestras consultas, ya no es como antes que podíamos hacerlo desde la URL. Recordemos que una inyección SQL puede llevarse a cabo desde varios métodos de entrada, como son la URL o los datos enviados por peticiones POST al servidor; este es un claro ejemplo del último caso.

Iniciemos observando el resultado al acceder al sitio web: vemos que este nos da la bienvenida anexando nuestro nombre de usuario y un par de signos de admiración.

![4]

Esto me indica que este puede ser un parámetro vulnerable, tal vez por detrás se realiza una consulta para determinar el usuario actual que accedió al servicio por medio de la variable **username** enviada por POST. Esto es solo una suposición, por lo que nuestro siguiente paso será interceptar una petición y ver si podemos efectuar inyecciones desde dicha variable (como lo hicimos para bypasear el panel de logueo):

![5]

Esta petición la enviamos al Repeater para efectuar diversas consultas. Una vez ahí, intentamos nuevamente la inyección de bypass:

    ' or 1=1-- -

Nota: Recuerda que en Burpsuite, y al enviar los datos por la petición POST, los espacios deben ser sustituidos por signos de más (+).

![6]

Vemos que logramos acceder al servidor. Ahora, nuestro primer paso en una inyección SQL es siempre ver el número de columnas de la tabla para encontrar un parámetro del que nos podamos aprovechar. Esto ya se ha explicado con anterioridad en otras máquinas, por lo que no nos detendremos a explicar detalladamente cada query.

Primero hacemos el UNION SELECT con varios números ascendentes (1,2,3...) hasta encontrar una respuesta que no devuelva error en el servidor. En este caso, vemos que la tabla contiene 4 columnas y que el parámetro 2 es vulnerable.

    ' union select 1,2,3,4-- -

![7]

Ahora sustituimos el 2 en la consulta con los datos que deseemos obtener, por ejemplo, el nombre de la base de datos:

    ' union select 1,database(),3,4-- -

![8]

La base de datos se llama **website**. Ahora obtenemos las tablas con las que cuenta dicha DB:

    ' union select 1,group_concat(table_name),3,4 from information_schema.tables where table_schema='website'-- -

Nota: El group_concat, recordemos, nos ayuda a agrupar todos los resultados que obtengamos sin necesidad de tener que listar uno por uno (con limit 1,1 por ejemplo) si es que llega a haber más de una tabla en la base de datos. En este caso solo hay una llamada **users**.

![9]

Ahora enumeramos las columnas de dicha tabla:

    ' union select 1,group_concat(column_name),3,4 from information_schema.columns where table_name_='users'-- -

Esta vez la consulta devuelve varias columnas, he ahí la importancia del group_concat. Vemos dos que son de nuestro interés: la de **username** y la de **password**:

![10]

Finalmente, listamos el contenido de ambas columnas para obtener los datos guardados:

    ' union select 1,group_concat(username,0x3a,password),3,4 from users-- -

![11]

Nota: Recuerda que la página web devuelve el resultado de la consulta seguido de dos signos de admiración. Ten mucho cuidado de no considerarlos como parte de la contraseña, la cual es:

    My_P@ssW0rd123

# Acceso por SSH
Si probamos estas credenciales en el servicio SSH, veremos que son válidas y ahora hemos ganado acceso al sistema.

![12]

Buscamos en el directorio actual del usuario **smokey** la flag de usuario, pero esta no se encuentra aquí:

![13]

Incluso intentamos enumerar el sistema en busca de vías para escalar privilegios, pero por ejemplo, el sudo no se puede ejecutar aquí:

![14]

Durante dicha enumeración me encontré con que hay dos usuarios en el sistema, cada uno con su propio directorio de trabajo: **smokey** (el actual) y **hazel**. De hecho, si entramos al directorio de **hazel** veremos que este usuario tiene la flag de usuario, por lo que suponemos que primero debemos hacer pivoting horizontal a este usuario y después buscar la forma de escalar privilegios.

![15]

# Pivoting horizontal por un descuido del usuario
Ahora, he aquí lo más importante que debemos saber: los usuarios son ingenuos y un poco despistados en ocasiones, y esta ingenuidad o descuido puede llegar a ser peligrosa. Por más que buscamos alguna credencial en el sistema, esta no aparecía. Ahora bien, ¿qué se debe hacer cuando no sabemos la credencial? Improvisar.

Como atacantes, debemos considerar que este usuario puede ser muy flojo para idearse una buena contraseña, por lo que probar con algunos clásicos no está mal (nunca subestimes esto ni lo dejes pasar, siempre prueba contraseñas por defecto o "simples"):

    admin
    administrator
    password
    password123

Pero algo que también debes probar como contraseña es el mismo nombre del usuario, e incluso puedes añadirle algunas variaciones como 123, etc. En este caso resultó ser este último caso: la contraseña del usuario **hazel** es **hazel**.

![16]

Como podemos ver, ya podemos ver la flag de usuario, la cual nos indica que hemos llegado hasta aquí gracias a SQLi y contraseñas débiles.

![17]

# Privilege Escalation
Si listamos los permisos sudo de nuestro usuario, veremos que podemos realizar un seteo de variables de entorno (SETENV) y la ejecución sin contraseña (NOPASSWD) de un script de python que hashea contraseñas o algo parecido.

![18]

Este es un escenario perfecto para un Python Library Hijacking, o secuestro de bibliotecas de Python. El objetivo es crear nosotros una biblioteca falsa (a la que el script que ejecutemos esté llamando) para que entonces el script original realice las operaciones que hemos definido en dicha biblioteca. Para ello, deberemos también modificar la variable de entorno PYTHONPATH que define las rutas desde donde Python puede encontrar las bibliotecas del lenguaje, de tal forma que apunte siempre a nuestra biblioteca falsa.

Si vemos el código fuente del script, veremos que manda llamar a la biblioteca **hashlib**:

![19]

Por lo tanto, nos movemos a un directorio donde tengamos permisos de escritura (en este caso /tmp) y crearmos una biblioteca falsa llamada **hashlib.py**, la cual contendrá el código a ejecutar de manera privilegiada. Sí, otra vez el SUID a la bash. Es lo más rápido, pero hay muchas otras formas, prometo cambiarlo para la siguiente.

    echo "import os;os.system('chmod +s /bin/bash')" > /tmp/hashlib.py

![20]

Ahora, solo debemos ejecutar como sudo un cambio de la variable de entorno PYTHONPATH y ejecutar el script como está definido en el sudoers:

    sudo PYTHONPATH=/tmp /usr/bin/python3 /home/hazel/hasher.py

Veremos que la ejecución lanza error, lo cual es normal. Sin embargo, si listamos los permisos de la bash veremos que ya es SUID:

![21]

Por último, migramos a la bash y vemos la flag del usuario root.

![22]

[1]:/assets/images/inoculation/1.png
[2]:/assets/images/inoculation/2.png
[3]:/assets/images/inoculation/3.png
[4]:/assets/images/inoculation/4.png
[5]:/assets/images/inoculation/5.png
[6]:/assets/images/inoculation/6.png
[7]:/assets/images/inoculation/7.png
[8]:/assets/images/inoculation/8.png
[9]:/assets/images/inoculation/9.png
[10]:/assets/images/inoculation/10.png
[11]:/assets/images/inoculation/11.png
[12]:/assets/images/inoculation/12.png
[13]:/assets/images/inoculation/13.png
[14]:/assets/images/inoculation/14.png
[15]:/assets/images/inoculation/15.png
[16]:/assets/images/inoculation/16.png
[17]:/assets/images/inoculation/17.png
[18]:/assets/images/inoculation/18.png
[19]:/assets/images/inoculation/19.png
[20]:/assets/images/inoculation/20.png
[21]:/assets/images/inoculation/21.png
[22]:/assets/images/inoculation/22.png