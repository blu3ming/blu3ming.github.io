---
layout: single
title: Máquina DevOops - HackTheBox (OSCP Style)
excerpt: "Resolución de la máquina DevOops, la cual figura en la plataforma con un nivel de dificultad Medium (aunque podría catalogarla como una máquina Easy)."
date: 2021-09-22
classes: wide
header:
  teaser: /assets/images/devoops/portada.png
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
	
	Solo hay dos puertos abiertos: el 22 (SSH) y el 5000
	
+ Si hacemos un escaneo más exhaustivo en busca de servicios y versiones que corren en dichos puertos, veremos lo siguiente:

	![2]
	
	El puerto 5000 resulta ser un servidor HTTP que corre con Gunicorn. No detecta nada de relevancia.
	
+ Si escaneamos con **whatweb**, no veremos información relevante que nos ayude en la fase de reconocimiento.

	![3]
	
+ Visitando el servicio web desde un navegador, nos encontraremos con un sitio web sin terminar, que solo renderiza una imágen y muestra la leyenda **Under Construction**. El código fuente no muestra nada que sea de utilidad.

	![4]

+ Sin más información con la cual trabajar, procedemos a hacer un fuzzing en busca de directorios ocultos. Cabe destacar que primero se hizo un escaneo con la diagonal al final de la palabra **FUZZ**:

	``wfuzz -c [...] http://10.10.10.91:5000/FUZZ/``
	
	Sin embargo, este escaneo no lograba encontrar nada, por lo que se intentó hacer el escaneo sin la diagonal final:
	
	``wfuzz -c [...] http://10.10.10.91:5000/FUZZ``
	
	Con esta variación, **wfuzz** logró detectar dos directorios en el sistema. Queda como experiencia y aprendizaje buscar siempre con ambas variantes.

	![5]
	
	El directorio **feed** solo contiene la imágen que se renderiza al visitar el index del sitio web. Sin embargo, llama la atención el directorio **upload**.
	
+ Visitando este directorio, veremos un panel de subida de archivos (al parecer XML). Este archivo, de acuerdo a lo que indica la página, deben contar con las etiquetas: **Author, Subject y Content**.

	![6]
	
	Debemos probar varias cosas. Primero debemos ver qué pasa con un archivo común y corriente que no es XML.
	
	![7]
	
	Este archivo no ocasiona nada en el sitio web, solo redirige al mismo sin mostrar información de error o éxito alguno. Si intentamos poner un archivo XML mal formado (es decir, sin las etiquetas solicitadas), veremos lo siguiente:
	
	![8]
	
	![9]
	
	Como podemos ver, esto ocasiona un error en el servidor, por lo que confirmamos que el sitio solo recibe archivos XML. Desde este momento se puede pensar en un ataque XXS, solo debemos corroborar si el sitio renderiza los archivos o no. Para ello, formamos un archivo **XML** con las etiquetas solicitadas:
	
	* Author
	* Subject
	* Content
	
	![10]
	
	Si subimos este archivo al sitio web, veremos que lo carga y renderiza su contenido. Es en este momento que podemos confirmar que un **XXE** es posible.
	
	![11]
	
	¿En qué consiste este ataque? Un **XXE** (o XML External Entity) son inyecciones de código efectuadas mediante una entidad, la cual es como una especie de variable que permitiría acceder a contenido local o remoto. Si la aplicación web renderiza contenido XML, procesará la entidad contenida y efectuará la instrucción solicitada, como puede ser un **LFI**.
	
	Declaramos una entidad como cabecera de la siguiente manera:
	
	```
		<!--?xml version="1.0" ?-->
		<!DOCTYPE replace [<!ENTITY ent SYSTEM "file:///etc/passwd"> ]>
	```
	
	De esta manera, declaramos una entidad que cargará un archivo del sistema, que será el /etc/passwd.
	
	El XML estará estructurado casi de manera similar, solo que en uno de los campos, deberemos introducir la entidad creada de esta manera:
	
	``&ent;``
	
	![12]
	
	Vemos que en la página web se renderiza el /etc/passwd del sistema, por lo que podemos corroborar que podemos listar archivos locales (LFI). Llama la atención la ruta donde se guardan estos archivos, la cual, según el sistema, es /home/roosa/depliy/src. Con esto podemos deducir que hay un usuario en el sistema llamado **roosa**, cuyo directorio de trabajo está ubicado en /home/roosa.
	
	![13]
	
	Con la ruta de trabajo del usuario **roosa** en mente, podemos intentar listar su **id_rsa** en caso de que esta exista, la cual debería estar en el directorio **.ssh**.
	
	![14]
	
	Vemos que la página nos devuelve la clave **id_rsa** del usuario, por lo que la pasamos a nuestra máquina de atacante.
	
	![15]
	
	![16]
	
	Dándole permisos **600**, nos intentamos conectar con esta clave. Como podemos ver, recibimos una conexión, por lo que la clave es válida y ya tenemos acceso al sistema.
	
	![17]
	
+ Ya dentro del sistema, podemos listar la flag de usuario.

	![18]
	
	El siguiente paso es escalar privilegios en el sistema. Se intentó enumerar lo siguiente:
	
	* Permisos SUID: FALSE
	* Tareas Cron: FALSE
	* Credenciales hardcodeadas en archivos de configuración: FALSE
	* Grupos: TRUE
	
	Se encontró que el usuario **roosa** formaba parte del grupo **adm**, el cual permite listar los logs del sistema. Muchas veces pueden existir credenciales hardcodeadas en los logs, por desgracia, no se encontró nada de relevancia.
	
	Llama la atención que el directorio de trabajo del usuario esté lleno de carpetas de trabajo, no muy comunes en máquinas HTB:
	
	![19]
	
	Si listamos ficheros dentro de este directorio de trabajo, veremos muchas cosas.
	
	``find . -type f 2>/dev/null``
	
	![20]
	
	Lo primero sería ir filtrando contenido. Vemos muchos archivos que provienen de una carpeta llamada **.local**, los cuales no parecen contener información relevante, así que los filtramos:
	
	``find . -type f 2>/dev/null | grep -v "\.local"``
	
	![21]
	
	Con esta información filtrada, vemos que ahora aparecen muchos directorios provenientes de la carpeta **work**. Sobre todo llama la atención que dentro haya un directorio git. Recordemos que dentro de estos directorios podemos encontrar información de repositorios, y como bien sabemos, muchas veces puede haber credenciales hardcodeadas dentro.
	
	Si listamos este directorio, veremos que efectivamente contiene un repositorio. Lo siguiente sería buscar los commits que se hayan hecho en busca de un nombre descriptivo que indique la eliminación de información sensible.
	
	![22]
	
	Primero, listamos los commits que se han hecho en el repositorio:
	
	``git log``
	
	![23]
	
	Llama la atención uno que tiene como descripción **"reverted accidental commit with proper key"**. Podemos ir deduciendo que en esta versión se añadió una clave o bien, se reemplazó una clave por otra.
	
	Si listamos los cambios que se hicieron en el repositorio durante dicho commit, veremos la información que estábamos buscando:
	
	``git log -p COMMIT``
	
	![24]
	
	La parte resaltada en rojo indica la información que fue eliminada, mientras que la parte en verde oscuro indica la información que fua añadida. ¿Por qué se eliminó una clave privada y se reemplazó con otra? Tal vez esta es la clave id_rsa de algún usuario dentro del sistema, y como solo nos queda el usuario **root**, no perdemos nada con intentarlo. Primero, nos traemos la clave a nuestra máquina:
	
	![25]
	
	Ya en nuestra máquina, le proporcionamos los permisos necesarios e intentamos conectarnos al sistema como el usuario **root**. Vemos que la clave es correcta y nos permite conectarnos sin problema alguno:
	
	![26]

	Por último, solo nos queda listar la flag de root. Con esto, hemos terminado de comprometer la máquina.
	
	![27]
	
[0]:/assets/images/devoops/0.png
[1]:/assets/images/devoops/1.png
[2]:/assets/images/devoops/2.png
[3]:/assets/images/devoops/3.png
[4]:/assets/images/devoops/4.png
[5]:/assets/images/devoops/5.png
[6]:/assets/images/devoops/6.png
[7]:/assets/images/devoops/7.png
[8]:/assets/images/devoops/8.png
[9]:/assets/images/devoops/9.png
[10]:/assets/images/devoops/10.png
[11]:/assets/images/devoops/11.png
[12]:/assets/images/devoops/12.png
[13]:/assets/images/devoops/13.png
[14]:/assets/images/devoops/14.png
[15]:/assets/images/devoops/15.png
[16]:/assets/images/devoops/16.png
[17]:/assets/images/devoops/17.png
[18]:/assets/images/devoops/18.png
[19]:/assets/images/devoops/19.png
[20]:/assets/images/devoops/20.png
[21]:/assets/images/devoops/21.png
[22]:/assets/images/devoops/22.png
[23]:/assets/images/devoops/23.png
[24]:/assets/images/devoops/24.png
[25]:/assets/images/devoops/25.png
[26]:/assets/images/devoops/26.png
[27]:/assets/images/devoops/27.png