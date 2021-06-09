---
layout: single
title: Introducción al Buffer Overflow - Ejercicios de Protostar: stack2
excerpt: "Continuamos esta introducción por el Buffer Overflow, ahora con con la resolución del ejercicio stack2, empleando todo lo que hemos aprendido hasta ahora."
date: 2021-06-9
classes: wide
header:
  teaser: /assets/images/bof-intro-stack2/portada.png
  teaser_home_page: true
categories:
  - Buffer Overflow
  - Blog
tags:
  - Blog
  - Buffer Overflow
  - Guía
  - Exploiting
---

+ Como nosotros trabajamos con una conexión SSH a la máquina, primero nos conectamos por medio del cmd de Windows:

	![1]
	
	Recordatorio: Las credenciales de la máquina son:
	
	```
		Para usuario normal user:user
		Para usuario root	root:godmode
	```
	
	Cabe destacar que esta vez necesitaremos entrar como usuario root para la creación de un script en Python.
	
+ Nuestro siguiente objetivo es el ejercicio **stack2**, el cual tiene el siguiente código fuente:
	
	```
	#include <stdlib.h>
	#include <unistd.h>
	#include <stdio.h>
	#include <string.h>
	 
	int main(int argc, char **argv)
	{
	  volatile int modified;
	  char buffer[64];
	  char *variable;
	  
	  variable = getenv("GREENIE");
	  
	  if(variable == NULL) {
		  errx(1, "please set the GREENIE environment variable\n");
	  }
	 
	  modified = 0;
	  
	  strcpy(buffer, variable);
	 
	  if(modified == 0x0d0a0d0a) {
		  printf("you have correctly modified the variable\n");
	  } else {
		  printf("Try again, you got 0x%08x\n", modified);
	  }
	}
	```
	
	**Nota:** Olvidé mencionar en la entrada anterior que todos los binarios a explotar se localizan en la carpeta **/opt/protostar/bin**
	
	Este código sigue una estructura similar al visto en **stack1**, ya que debemos setear la variable **modified** a un valor específico, siendo ahora **0x0d0a0d0a**. Lo que hace es obtener primero este valor de una variable de entorno en el sistema llamada **GREENIE**, la cual no es una variable por defecto en Linux. Esto quiere decir que primero debemos crearla, lograr almacenar en ella la cadena que ocasionará el desbordamiento y simplemente ejecutar el binario.
	
	Comencemos por lo básico, si no creamos la variable de entorno antes de ejecutar el binario, este nos lo informará con el siguiente mensaje:
	
	![2]
	
	Con esto en mente, ahora crearemos la variable de entorno. En Linux, se hace por medio del siguiente comando:
	
	``export GREENIE=AAAA``
	
	Como parte de la prueba, solo se le ha dado el valor de 4 letras A a la variable. Veamos cómo se comporta el binario ante esto.
	
	![3]
	
	Como podemos observar, de nueva cuenta nos indica que no se ha ejecutado correctamente, y nos imprime en pantalla el valor actual de la variable **modified**, la cual recordemos, debemos lograr sobreeescribir al desbordar la variable **buffer**. Intentemos ahora ejecutar el binario, seteando primero la variable de entorno con el tamaño máximo de la variable **buffer** (64 bytes). Para ello, empleamos de nueva cuenta Python:
	
	![4]
	
	Si ahora, de nueva cuenta lo intentamos con un valor más (65 bytes), veremos cómo la variable se desborda y comenzamos a sobreescribir **modified**
	
	![5]
	
	Viendo cómo podemos sobreescribir la variable, ahora solo nos queda hacerlo con el valor indicado (0x0d0a0d0a). Para ello, buscamos a qué caracteres pertenecen en ASCII; para ello, empleamos el manual de ASCII de Linux:
	
	``man ascii``
	
	![6]
	
	Como podemos ver, corresponden a un salto de línea y un retorno, respectivamente. Si intentamos colocar primero un salto de línea en la parte de desbordamiento, veremos que la variable no se sobreescribe, por lo que este enfoque ya no nos es útil por ahora.
	
	![7]
	
	Para llevar a cabo la explotación, nos valdremos de un script en Python. Este es el código:
	
	```
		import struct
		buffer = "A"*64
		pad = struct.pack("I",0x0d0a0d0a)
		print buffer+pad
	```
	
	Este código es muy simple. Se basa en imprimir todo el string que usaremos para desbordar el buffer de memoria. Primero creamos la variable **buffer**, la cual contiene los 64 caracteres "A" que ya conocemos y hemos venido manejando. Lo nuevo está en la variable **pad**. Esta variable emplea la biblioteca **struct** para crear una cadena en big-endian (**I**, si fuéramos a emplear little-endian se escribiría **<I**) del string en hexadecimal 0x0d0a0d0a. Por último, solo imprimimos la unión de ambos.
	
	Lo que realiza la biblioteca **struct** a bajo nivel, es crear un objeto de bytes que contiene los valores especificados, en este caso 0x0d0a0d0a, "empaquetados" de acuerdo al formato proporcionado como primer argumento, en este caso la letra **I** que implica un formato en big-endian.
	
	Como el resultado del script es una cadena de caracteres con todo lo que necesitamos para realizar el desbordamiento, al ejecutarlo veremos lo siguiente:
	
	![8]
	
	Vemos esos espacios en blanco debajo de las "A" en un intento de representar los saltos de línea y retornos que, como vimos antes, corresponden a dicho código en hexadecimal.
	
	**Nota:** Para poder tener permisos de escritura dentro de la carpeta y poder realizar el script, es necesario realizar un cambio de usuario a **root** si es que no te logueaste como este usuario antes.
	
	Por último, solo nos resta pasarle la salida del script a la variable de entorno **GREENIE**, y ejecutar el binario **stack2**.
	
	![9]
	
	Como se puede observar, la salida del programa ha cambiado, indicándonos que hemos logrado sobreescribir la variable con el valor correcto, habiendo finalizado correctamente el ejercicio.
	
	En la siguiente entrada, veremos cómo resolver el siguiente ejercicio de esta serie, el cual es **stack3**.
    
[1]:/assets/images/bof-intro-stack2/1.png
[2]:/assets/images/bof-intro-stack2/2.png
[3]:/assets/images/bof-intro-stack2/3.png
[4]:/assets/images/bof-intro-stack2/4.png
[5]:/assets/images/bof-intro-stack2/5.png
[6]:/assets/images/bof-intro-stack2/6.png
[7]:/assets/images/bof-intro-stack2/7.png
[8]:/assets/images/bof-intro-stack2/8.png
[9]:/assets/images/bof-intro-stack2/9.png