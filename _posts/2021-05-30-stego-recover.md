---
layout: single
title: StegoRecover - Un script para recuperar la contraseña de steghide
excerpt: "StegoRecover es un script que permite recuperar, por medio de fuerza bruta, la contraseña con la que se ocultó información en una imagen empleando la herramienta steghide."
date: 2021-05-30
classes: wide
header:
  teaser: /assets/images/stego-recover/portada.png
  teaser_home_page: true
categories:
  - Scripts
tags:
  - Scripts
  - BruteForce
  - Steghide
  - Esteganografía
---

	![1]
	
StegoRecover es una herramienta programada en Python que permite cargar un archivo con datos ocultos (esteganografía) con **steghide**, más un diccionario de posibles contraseñas, y mediante un ataque de fuerza bruta, recuperar la contraseña de dicho archivo.

+ **¿Qué es Steghide?**

	Steghide es un programa de esteganografía que permite esconder (ocultar) datos o información en una amplia variedad de tipos de imágenes. Permite ocultar información empleando o no una contraseña, la cual permitirá al destinatario extraer la información que se haya ocultado.
	
	![2]
	
	Si por un casual, hemos perdido u olvidado la contraseña con la que se ocultó la información, necesitamos recuperarla de algún modo, puesto que no se podrán extraer dichos datos de ninguna forma.
	
	Es ahí donde entra este script, el cual automatiza la fase de fuerza bruta para encontrar la contraseña perdida.
	
+ **Origen**
	
	Este script nace de un intento por automatizar la recuperación de la contraseña para el archivo "cute-alien.jpg" de la máquina **Agent Sudo** de **TryHackMe**, la cual requiere de una contraseña para extraer la información oculta y poder seguir con la explotación de la máquina.
	
+ **Uso**

	``python stegorecover.py steg_file wordlist``
	
	El script se ejecuta mandando llamar a Python, pasándole la ruta del archivo que contiene la información oculta (steghide) y la ruta del diccionario donde se encuentre la contraseña.
    
	![3]
	
	Muestra del uso incorrecto de la herramienta, donde se le muestra al usuario cómo ejecutarla correctamente.

	![4]
	
+ **¿Cómo funciona?**

	El script ejecuta a nivel de sistema el comando:
	
	``steghide extract -sf STEG_FILE -p PASSWORD``
	
	Con este comando, mandamos a llamar a la herramienta **steghide** con la bandera de extracción, añadiendo el archivo con la información oculta y una posible contraseña. El script lee el diccionario y pasa una por una las contraseñas para que se ejecute este comando.
	
	Si la ejecución del comando no es exitosa, es decir, la contraseña proporcionada no es la correcta, el script continua con la siguiente y así hasta terminar el diccionario por completo.
	
	![5]
	
	Si encuentra la contraseña correcta, y dado que ejecurta por detrás la extracción de información con **steghide**, este proceso se lleva a cabo correctamente, extrayendo la información oculta e indicándole al usuario dónde quedó guardada.
	
	![6]
	
	**Nota: **Cabe destacar que, como el script ejecuta a nivel de sistema la herramiente **steghide**, es necesario que esta esté instalada en el sistema primero. Para hacerlo, basta con ejecutar en Linux el comando:
	
	``sudo apt install steghide``
	
+ **Código del script**

	```
		#!/usr/bin/python

		import commands
		import sys, os, signal
		import time
		from pwn import *

		def def_handler(sig, frame):
			log.failure("Exiting...")
			sys.exit(1)

		signal.signal(signal.SIGINT, def_handler)

		print("\n")
		log.info("StegoRecover (by blu3ming)")
		print(r"""
								 .       .
								/ `.   .' \
						.---.  <    > <    >  .---.
						|    \  \ - ~ ~ - /  /    |
						 ~-..-~             ~-..-~
					 \~~~\.'                    `./~~~/
					  \__/                        \__/
					   /                  .-    .  \
				_._ _.-    .-~ ~-.       /       }   \/~~~/
			_.-'q  }~     /       }     {        ;    \__/
		   {'__,  /      (       /      {       /      `. ,~~|   .     .
			`''''='~~-.__(      /_      |      /- _      `..-'   \\   //
						/ \   =/  ~~--~~{    ./|    ~-.     `-..__\\_//_.-'
					   {   \  +\         \  =\ (        ~ - . _ _ _..---~
					   |  | {   }         \   \_\
					  '---.o___,'       .o___,'       -r.millward-""")
		print("\n")

		if len(sys.argv) != 3:
			print("\n")
			log.info("Usage: python " + str(sys.argv[0]) + " steg_file wordlist")
			print("\n")
			log.failure("Exiting...")
			sys.exit(1)

		else:
			line_count = sum(1 for line in open(sys.argv[2]))
			log.info("%d Passwords loaded" % line_count)
			time.sleep(2)

			p1 = log.progress("Recovering steg password...")
			p2 = log.progress("Password")
			time.sleep(2)
			with open(str(sys.argv[2])) as wordlist:
				for password in wordlist:
					p2.status("%s" % password)
					rc, out = commands.getstatusoutput("steghide extract -sf " + str(sys.argv[1]) + " -p " + str(password) + " >/dev/null 2>&1")
					if "wrote extracted" in out:
						p1.success("Password found!")
						p2.success("%s" % password)
						print("\n")
						log.info("%s" % out)
						wordlist.close()
						sys.exit(0)

			p1.failure("Password not found!")
			print("\n")
			log.info("Password not found in wordlist file... Exiting...")
			wordlist.close()
			sys.exit(1)
	```

	Página de GitHub del repositorio

	[https://github.com/blu3ming/StegoRecover](https://github.com/blu3ming/StegoRecover)
	
[1]:/assets/images/stego-recover/1.png
[2]:/assets/images/stego-recover/2.png
[3]:/assets/images/stego-recover/3.png
[4]:/assets/images/stego-recover/4.png
[5]:/assets/images/stego-recover/5.png
[6]:/assets/images/stego-recover/6.png