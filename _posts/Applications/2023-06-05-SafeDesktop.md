---
title: SafeDesktop - Backup de configuracion escritorio Gnome
date: 2023-06-05 10:00:00 +0100
categories: [Applications]
tags: [software, tips]     # TAG names should always be lowercase
img_path: /assets/pictures/
author: <noah>
---
## Añadir accesos directos de Office Online a Gnome

Cuando hacemos una instalación limpia en el equipo siempre lleva su tiempo dejar el escritorio a nuestro gusto. En mi caso que tengo el equipo fijo y el portátil, me gusta que la configuración en los dos sea la misma, por eso cuando descubrí esta aplicación me pareció muy interesante.

En el centro de software de Gnome está disponible como paquete flatpak:

![savedesktop](savedesktop.png)

También podemos instalar con el siguiente comando: 
``` bash
flatpak install flathub io.github.vikdevelop.SaveDesktop
```

Por útlimo sólo queda configurar:

![savedesktop-1](savedesktop-1.png)

En caso de formateo o instalación limpia podemos restaurar el backup que hemos realizado con SaveDesktoop. 

(NOTA: me queda pendiente realizar la prueba de restauración para ver cual es el resultado real del backup, aunque como realizo los backups semanales con deja-dup es posible que no sea necesario hacer uso de esta aplicación)

Ficheros:  
[Iconos de Office](/assets/files/icons.zip)  
[Ficheros .desktop](/assets/files/links.zip)



***
Fuentes:  
[https://thecheis.com/2023/06/04/guarda-configuracion-escritorio-savedesktop/](https://thecheis.com/2023/06/04/guarda-configuracion-escritorio-savedesktop/)

