---
layout: single
title: Room Hacking with Powershell - TryHackMe (Offensive Pentesting Path)
excerpt: "Esta room es un laboratorio de práctica para aprender comandos y scripting de Powershell, entendiendo cómo funciona su sintaxis y cómo nos puede ayudar en la fase de enumeración."
date: 2022-07-20
classes: wide
header:
  teaser: /assets/images/powershell/portada.png
  teaser_home_page: true
categories:
  - Comunidad
  - Blog
  - Writeup
tags:
  - powershell
  - scripting
  - windows
---

# Introducción
La room se encuentra en [TryHackMe](https://tryhackme.com/room/powershell). Se trata de una máquina Windows donde tendremos que superar una serie de desafíos encontrando archivos y contenido con ayuda de comandos en Powershell. Esta será más una guía de la room que un walkthrough en sí, funcionando más como un cheatsheet de este.

Por cierto, nos hemos saltado toda la sección de Active Directory ya que no son máquinas que resolver como tal, sino guías para entender cómo enumerar y los tipos de ataque que se presentan en este tipo de entornos; la misma guía que da TryHackMe para cada room es suficiente y no vi necesario hacer entradas al respecto.

# Índice
- [Introducción](#introducción)
- [Índice](#índice)
- [Basic Powershell Commands](#basic-powershell-commands)
  - [What is the location of the file "interesting-file.txt"](#what-is-the-location-of-the-file-interesting-filetxt)
  - [Specify the contents of this file](#specify-the-contents-of-this-file)
  - [How many cmdlets are installed on the system(only cmdlets, not functions and aliases)?](#how-many-cmdlets-are-installed-on-the-systemonly-cmdlets-not-functions-and-aliases)
  - [Get the MD5 hash of interesting-file.txt](#get-the-md5-hash-of-interesting-filetxt)
  - [What is the command to get the current working directory?](#what-is-the-command-to-get-the-current-working-directory)
  - [Does the path "C:\Users\Administrator\Documents\Passwords" Exist(Y/N)?](#does-the-path-cusersadministratordocumentspasswords-existyn)
  - [What command would you use to make a request to a web server?](#what-command-would-you-use-to-make-a-request-to-a-web-server)
  - [Base64 decode the file b64.txt on Windows.](#base64-decode-the-file-b64txt-on-windows)
- [Enumeration](#enumeration)
  - [How many users are there on the machine?](#how-many-users-are-there-on-the-machine)
  - [Which local user does this SID(S-1-5-21-1394777289-3961777894-1791813945-501) belong to?](#which-local-user-does-this-sids-1-5-21-1394777289-3961777894-1791813945-501-belong-to)
  - [How many users have their password required values set to False?](#how-many-users-have-their-password-required-values-set-to-false)
  - [How many local groups exist?](#how-many-local-groups-exist)
  - [What command did you use to get the IP address info?](#what-command-did-you-use-to-get-the-ip-address-info)
  - [How many ports are listed as listening?](#how-many-ports-are-listed-as-listening)
  - [What is the remote address of the local port listening on port 445?](#what-is-the-remote-address-of-the-local-port-listening-on-port-445)
  - [How many patches have been applied?](#how-many-patches-have-been-applied)
  - [When was the patch with ID KB4023834 installed?](#when-was-the-patch-with-id-kb4023834-installed)
  - [Find the contents of a backup file.](#find-the-contents-of-a-backup-file)
  - [Search for all files containing API_KEY](#search-for-all-files-containing-api_key)
  - [What command do you do to list all the running processes?](#what-command-do-you-do-to-list-all-the-running-processes)
  - [What is the path of the scheduled task called new-sched-task?](#what-is-the-path-of-the-scheduled-task-called-new-sched-task)
  - [Who is the owner of the C:](#who-is-the-owner-of-the-c)
- [Basic Scripting Challenge](#basic-scripting-challenge)
  - [What file contains the password? and What is the password?](#what-file-contains-the-password-and-what-is-the-password)
  - [What files contains an HTTPS link?](#what-files-contains-an-https-link)
- [Intermediate Scripting](#intermediate-scripting)

Basic Powershell Commands
==================================================================================================================
## What is the location of the file "interesting-file.txt"

Para responder esta pregunta, debemos ejecutar un comando que busque en el sistema por archivos:

    Get-ChildItem -Path 'C:\' -Recurse -ErrorAction SilentlyContinue | Where-Object -Property Name -Match 'interesting-file.txt'

Con esto le indicamos que busque en el directorio C: de manera recursiva e ignorando los errores de acceso que se pudiera encontrar un archivo cuyo nombre sea el indicado.

## Specify the contents of this file

Para obtener el contenido de un archivo empleamos el comando:

    Get-Content 'C:\Program Files\interesting-file.txt.txt'

![1]

## How many cmdlets are installed on the system(only cmdlets, not functions and aliases)?

El comando que nos devuelve el listado de cmdlets instalados en el sistema es:

    Get-Command

Sin embargo este nos devolverá todo lo que esté instalado, y la pregunta especifica que solo quiere los cmdlets. Con **Get-Member** obtenemos los parámetros que 
podemos solicitar del comando principal.

![2]

El parámetro que nos permitirá filtrar por lo que buscamos es **CommandType**. Solo le indicamos a la salida del comando anterior que busque por los tipos cmdlet únicamente:

    Get-Command | Where-Object {$_.CommandType -eq 'cmdlet'} | Measure

**Measure** nos permite contar todos los resultados obtenidos.

![3]

## Get the MD5 hash of interesting-file.txt

Buscamos por comandos que contengan en su nombre *hash*:

    Get-Command *hash*

![4]

La herramienta que nos ayudará es **GetFileHash**. Mandamos llamar a **Get-Help** para ver cómo podemos indicarle el algoritmo de hasheo. Esto se realiza con el parámetro **-Algorithm**

![5]

Por lo tanto, ejecutamos:

    Get-FileHash -Algorithm MD5 'C:\Program Files\interesting-file.txt.txt'

![6]

## What is the command to get the current working directory?

Buscamos por comandos que contengan en su nombre *location*. El comando que buscamos es:

    Get-Location

![7]

## Does the path "C:\Users\Administrator\Documents\Passwords" Exist(Y/N)?

Para saber si existe el directorio, solo intentamos hacer un *cd* a este (ignoro si este paso se realizaba de un modo diferente). Como podemos observar, no podemos acceder dado que no existe.

![8]

## What command would you use to make a request to a web server?

Buscamos por comandos que contengan en su nombre *reqest* y *web*. El comando que buscamos es:

    Invoke-WebRequest

De hecho, ya lo hemos estado utilizando en máquinas Windows.

![9]

## Base64 decode the file b64.txt on Windows. 

Proimero pasamos a una variable el contenido del archivo especificado:

    $content = Get-Content 'C:\Users\Administrator\Desktop\b64.txt'

Posteriormente, ejecutamos la decodificación de base64:

    $DECODED = [System.Text.Encoding]::ASCII.GetString([System.Convert]::FromBase64String($content))

Y por último, solo imprimimos la variable $DECODED:

    $DECODED

![10]

Enumeration
==================================================================================================================

## How many users are there on the machine?

El comando que nos devuelve todos los usuarios de una máquina es:

    Get-LocalUser

Recordemos que con **Measure** obtenemos la cantidad de resultados obtenidos por el comando anterior.

![11]

## Which local user does this SID(S-1-5-21-1394777289-3961777894-1791813945-501) belong to?

Seguimos empleando el mismo comando para listar a los usuarios, solo que ahora empleamos la propiedad *SID* para buscar a aquél que coincida con el indicado:

    Get-LocalUser | Where-Object -Property SID -Match 'S-1-5-21-1394777289-3961777894-1791813945-501'

![12]

## How many users have their password required values set to False?

Siguiendo el mismo principio que el ejercicio anterior, ahora empleamos el parámetro *PasswordRequired* y listamos aquellos que lo tengan seteado en *false*:

    Get-LocalUser | Where-Object -Property PasswordRequired -Match 'false'

![13]

## How many local groups exist?

Para listar los grupos locales empleamos:

    Get-LocalGroup

![14]

## What command did you use to get the IP address info?

Buscamos un comando que contenga la cadena IP en el nombre:

![15]

El comando que buscamos es:

    Get-NetIPAddress

Corroboramos el output:

![16]

## How many ports are listed as listening?

Este comando fue engañoso, dado que no contiene la cadena *ports* en su nombre, así que tuve que Googlearlo en su lugar:

    Get-NetTCPConnection

![17]

Si solo queremos filtrar aquellos que estén con estatus *Listen*:

    Get-NetTCPConnection | Where-Object -Property State -Match 'Listen'

## What is the remote address of the local port listening on port 445?

Solo damos un vistazo a la salida del comando anterior para resolverlo.

    ::

## How many patches have been applied?

De igual manera, este comando tuve que Googlearlo ya que no hay una cadena en su nombre que incluya *patch* o algo similar:

    Get-HotFix

![18]

## When was the patch with ID KB4023834 installed?

Filtramos la salida anterior en busca de un *HotFixID* con el valor especificado:

    Get-HotFix | Where-Object -Property HotFixID -Match 'KB4023834'

![19]

## Find the contents of a backup file.

Para buscar por un archivo backup, tenemos que buscar aquellos que contengan la cadena *.bak* en su nombre, y para ello nuevamente hacemos uso de *Get-ChildItem*:

    Get-ChildItem -Path 'C:\' -Recurse -ErrorAction SilentlyContinue -Include *.bak* -File

![20]

## Search for all files containing API_KEY

De manera similar al anterior, ahora debemos buscar por un archivo que contenga la cadena *API_KEY* dentro de su contenido, y para ello, hacemos uso de *Select-String*

    Get-ChildItem -Path 'C:\' -Recurse -ErrorAction SilentlyContinue | Select-String -pattern API_KEY

Esto nos devolverá un output bastante extenso sobre archivos que contengan dicha cadena, pero el que buscamos aparecerá casi al final del escaneo y será visible al momento; una API Key.

![21]

## What command do you do to list all the running processes?

Listamos comandos que contengan la cadena *process* en su nombre; el que buscamos es el siguiente:

    Get-Process

![22]

## What is the path of the scheduled task called new-sched-task?

No tengo una solución para este ejercicio, dado que el formato de respuesta en TryHackMe ya nos dice que es simplemente una diagonal, refiriéndose a la raíz:

    /

## Who is the owner of the C:

De manera análoga a cmd, busco un comando que contenga en su nombre la cadena *ACL*, que es la encargada de permisos en Windows; el comando que necesitamos es:

    Get-Acl 'C:\'

![23]

Basic Scripting Challenge
==================================================================================================================
El ejercicio nos pide modificar el script que encontraremos en el Escritorio (listening-ports.ps1) para poder responder a las preguntas planteadas, sin embargo, estas se responden simplemente con un comando oneliner.

## What file contains the password? and What is the password?

Buscamos un archivo que en su contenido contenga la cadena *password*, similar al ejercicio del apartado anterior:

    Get-ChildItem -Path 'C:\Users\Administrator\Desktop\emails' -Recurse -ErrorAction SilentlyContinue | Select-String -pattern password

Obtenemos respuesta para las primeras dos preguntas con solo ese comando.

![25]

## What files contains an HTTPS link?

De igual manera, ahora solo buscamos por la cadena *https*:

    Get-ChildItem -Path 'C:\Users\Administrator\Desktop\emails' -Recurse -ErrorAction SilentlyContinue | Select-String -pattern https

![26]

Intermediate Scripting
==================================================================================================================
El ejercicio nos solicita un script que sea capaz de escanear puertos en el localhost y determinar si estos se encuentran abiertos o no; para corroborar su funcionamiento, solicitan saber cuántos se encuentran abiertos en el localhost en el rango de 130 a 140. Para ello, empleé el comando *Test-NetConnection*, el cual entabla una conexión TCP con el host y puerto que le especifiquemos:

    Test-NetConnection localhost -p 80

Para resolver el ejercicio, primero declaro tres variables: host, puerto inicial y puerto final (el escaneo se lleva a cabo en un rango de puertos). Posteriormente realizo un bucle *for* para mandar llamar a el comando especificado con cada uno de los puertos dentro del rango declarado. Este comando devuelve varios valores, sin embargo, me centré primero en el llamado *TcpTestSucceeded*, el cual devuelve True si logró establecer una conexión.

![27]

Este, al ser ejecutado, devuelve que solo hay un puerto abierto, el 135:

![28]

Sin embargo, hay algo que no estoy tomando en consideración, y es el hecho de que el puerto puede estar abierto pero ser inaccesible, y para corroborarlo, debemos emplear otro de los resultados que devuelve el comando llamado *PingSucceeded*. Este nos indicará si el puerto está a la escucha y ha respondido, pero probablemente no sea capaz de establecer una conexión (razón por la que el TcpTestSucceeded pueda devolver False cuando el puerto sí está a la escucha). De esta manera, empleo ambas posibilidades en una operación OR dentro del *if*; adicional a ello, declaro un contador que me ayude a determinar cuántos puertos resultaron abiertos al final:

![29]

Si ejecutamos el script, veremos que para todos los puertos que antes habíamos considerado como cerrados, en realidad existen y responden al ping, pero son inaccesibles. Mientras que el 135, al responder al *TcpTestSucceeded*, sí lo es.

Por lo tanto, se concluye que los 11 puertos escaneados (entre el 130 y el 140) se encuentran abiertos.

![30]

El script resultante puedes consultarlo en mi [GitHub](https://github.com/blu3ming/PortScan-Powershell).

[1]:/assets/images/powershell/1.png
[2]:/assets/images/powershell/2.png
[3]:/assets/images/powershell/3.png
[4]:/assets/images/powershell/4.png
[5]:/assets/images/powershell/5.png
[6]:/assets/images/powershell/6.png
[7]:/assets/images/powershell/7.png
[8]:/assets/images/powershell/8.png
[9]:/assets/images/powershell/9.png
[10]:/assets/images/powershell/10.png
[11]:/assets/images/powershell/11.png
[12]:/assets/images/powershell/12.png
[13]:/assets/images/powershell/13.png
[14]:/assets/images/powershell/14.png
[15]:/assets/images/powershell/15.png
[16]:/assets/images/powershell/16.png
[17]:/assets/images/powershell/17.png
[18]:/assets/images/powershell/18.png
[19]:/assets/images/powershell/19.png
[20]:/assets/images/powershell/20.png
[21]:/assets/images/powershell/21.png
[22]:/assets/images/powershell/22.png
[23]:/assets/images/powershell/23.png
[25]:/assets/images/powershell/25.png
[26]:/assets/images/powershell/26.png
[27]:/assets/images/powershell/27.png
[28]:/assets/images/powershell/28.png
[29]:/assets/images/powershell/29.png
[30]:/assets/images/powershell/30.png