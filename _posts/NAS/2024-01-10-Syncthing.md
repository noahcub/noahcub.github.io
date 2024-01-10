---
title: Instalación de Syncthing en Unraid y Debian
date: 2024-01-10 09:00:00 +0100
categories: [Nas]
tags: [software, config, unraid]     # TAG names should always be lowercase
img_path: /assets/pictures/Nas
author: <noah>
---
## Instalación de Syncthing en Unraid y Debian

## Unraid

La instalación se realizará con el docker de Syncthing que nos aparece en la búsqueda de aplicaciones del respositorio linuxserver's Repository:

Yo he utilizado todos los datos que carga por defecto con especial atención a estos dos:

Appdata: /mnt/user/appdata/syncthing

Path: /data1: /mnt/user/carpeta_que_queremos_tenga_acceso_el_docker_Syncthing

Es muy importante el datos que muestra en este punto: ***Container Path: /data1 Data1*** porque es la ruta de la carpeta dentro de syncthing.

![syncthing.png](syncthing.png)

Una vez instalado accedemos al webUI de la aplicación.  
Nos dirá que pongamos un password al usuario y comienza la configuración.

![syncthing-2.png](syncthing-2.png)

En la esquina inferior derecha añadimos los dispositivos de confianza.  

En la parte izquierda configuramos las carpetas que vamos a compartir.

![syncthing-3.png](syncthing-3.png)

En esta carpeta de ejemplo se ve como debe ser las rutas dentro del contenedor. En mi caso: Debemos empezar con /data1/ y nos irá desplegando las carpetas donde hemos montado el volumen del contenedor:

```bash
/data1/Backups/NAS
```
***Importante:  Identificador requerido para la carpeta. Debe ser el mismo en todos los dispositivos del clúster.***  
Este identificador nos sirve para sincronizar carpetas entre distintos dispositivos y ubicarlas donde queramos.

En mi caso, en la pestaña avanzado he marcado la opción "solo enviar" para guardar el backup en otra ubicación y que esta carpeta no pueda ser modificada.

![syncthing-4.png](syncthing-4.png)


## Debian

En Debian 12 se instala con apt porque syncthing ya está en los repositorios. Según la web de Syncthing tenemos que importar la clave y añadir repositorio pero ya que un sencillo ***apt install syncthing*** es suficiente:

```bash
sudo apt install syncthing
```
A continuación creamos un servicio systemd:

```bash
sudo systemctl enable syncthing@mi_usuario.service
sudo systemctl start syncthing@mi_usuario.service

sudo systemctl status syncthing@noe.service
```

En este dispositivo la configuración es similar, pero es necesario tener algunas consas en cuenta, ya que en mis equipos estoy conectado continuamente por VPN al servidor Unraid y me he llevado alguna sorpresa al sincronizar gran cantidad de datos a través de la red movil. 

Syncthing tiene predeterminado el reinicio cuando está arrancado entonces aunque hagamos un
```bash
sudo systemctl stop syncthing@mi_usuario.service
```
volverá a arrancar nuevamente.  

Para solucionarlo vamos a Opciones Avanzadas y desmarcamos ***Restart On Wakeup***

![syncthing-5.png](syncthing-5.png)

Con esto cuando paremos el servicio con ***sudo systemctl stop syncthing@mi_usuario.service*** no volverá a arrancar.



***   
Fuentes y enlaces de interés que ayudaran a complementar esta guía:  

[https://forum.syncthing.net/t/syncthing-restarts-after-sleep-mode-and-rescans-disable/8992](https://forum.syncthing.net/t/syncthing-restarts-after-sleep-mode-and-rescans-disable/8992)  
[https://www.linuxfordevices.com/tutorials/ubuntu/syncthing-install-and-setup](https://www.linuxfordevices.com/tutorials/ubuntu/syncthing-install-and-setup)  

