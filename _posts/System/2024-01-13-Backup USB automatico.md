---
title: Backup carpetas USB automaticamente al conectar al equipo
date: 2024-01-13 08:00:00 +0100
categories: [System]
tags: [Apps, software, debian]     # TAG names should always be lowercase
author: <noah>
---


## Realizar un backup del pendrive al conectar al equipo 

**Creamos el script de backup con los comandos de rsync**

Con este pequeño script  hacemos el backup del USB del trabajo a nuestro disco duro:  
	Origen:  /ruta origen  
	Destino: /ruta destino  
	OJO - cuando tenemos espacios en los nombres hay que usar '\' para que el script sepa que es un espacio  

``` bash
#!/bin/bash
<< 'Comment'
	Opciones rsync: 
	--delete: borrar los archivos que fueron borrados de la carpeta origen en la carpeta destino
	-a: mantiene el usuario, grupo, permisos, fecha y hora, así como los enlaces simbólicos
	-v: la información imprimida durante la ejecución del programa será mucho más detallada,
	-h: nos da las tasas de transferencia y el tamaño de los archivos en unidades razonables
	-u: evitar copiar archivos que ya tenemos en la carpeta de destino que no has sido modificados en la carpeta de origen
Comment

rsync --progress --stats --delete  -avhiu /media/user/Nuevo\ vol/Origen /home/user/Destino


```

**Creamos la regla udev para que se ejecute nuestro script al conectar el usb**  
Primero debemos obtener el idProduct y el idVendor del nuestro usb
``` bash
lsusb
Bus 002 Device 003: ID 0781:5581 SanDisk Corp. Ultra
Bus 002 Device 001: ID 1d6b:0003 Linux Foundation 3.0 root hub
```
idVendor = 0781  
idProduct = 5581  

Con estos datos creamos un fichero .rules:
``` bash
sudo nano /etc/udev/rules.d/10.backup_usb.rules
```
``` bash
SUBSYSTEM=="block", ACTION=="add", ATTRS{idVendor}=="0781", ATTRS{idProduct}=="5581", SYMLINK+="external", RUN+="/home/user/backup_usb.sh"
```

Por último, recargamos las reglas udev para que el sistema ejecute el script al enchufar nuestro pendrive:  
``` bash
udevadm control --reload
```
