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
Podemos instalar el icono de la bandeja del sistema para tener acceso directo a la aplicación al mismo tiempo que con colores indica el estado de sincronización de syncthing.

```bash
sudo apt install syncthingtray
```

Para iniciar el servicio tenemos dos opciones: crear un servicio en systemd o iniciarlo con la aplicación de syncthingtray.  
En el portatil lo he configurado con servicio systemd, pero en el fijo por algún problema syncthingtray no detectaba que estaba el servicio corriendo y he decidido hacerlo mediante el Startup Applications como cualquier otra aplicación.  

## Servicio Syncthing con Systemd  

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

## Inicio mediante Syncthingtray  
Muy sencillo de configurar:
![syncthing-8.png](syncthing-8.png)

![syncthing-9.png](syncthing-9.png)



## Problemas de sincronización  

En el equipo de oficina he tenido un problema con la sincronización de las carpetas cuando en el NAS la marco como "Solo Enviar" y en el PC como "Solo Recibir" con la intención de hacer un backup de datos que me interesan sin que se puedan modificar por error en el equipo destino. Pues con esta configuración, syncthing me marcaba el siguiente error en el equipo con la carpetas en "Solo Recibir" como se ve en la imagen.

![syncthing-6.png](syncthing-6.png)

Yo creo que es posible que el error se deba a que esa carpeta está en una partición NTFS y algo falla con los permisos de las carpetas, ya que en el portátil no he tenido ningún problema con ese error.
Leyendo en varios foros vi que marcando en las opciones de las carpetas "Ignorar permisos" pudiera ser una solución.   

![syncthing-7.png](syncthing-7.png)

En principio parecía que no se solucionaba, pero siguiendo en los foros un usuario decía que reiniciando la base de datos de syncthing en el ordenador ya le funcionaba.

```bash
syncthing --reset-database
```
Yo hice lo mismo y en principio parece que funciona.


***   
Fuentes y enlaces de interés que ayudaran a complementar esta guía:  

[https://forum.syncthing.net/t/syncthing-restarts-after-sleep-mode-and-rescans-disable/8992](https://forum.syncthing.net/t/syncthing-restarts-after-sleep-mode-and-rescans-disable/8992)  
[https://www.linuxfordevices.com/tutorials/ubuntu/syncthing-install-and-setup](https://www.linuxfordevices.com/tutorials/ubuntu/syncthing-install-and-setup)  

