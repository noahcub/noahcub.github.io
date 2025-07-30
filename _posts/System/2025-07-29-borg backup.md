---
title: Backups con Borg, borgmatic y Vorta
date: 2025-07-29 10:00:00 +0100
categories: [System]
tags: [terminal, software, linux, vps, fedora]     # TAG names should always be lowercase
img_path: /assets/pictures/Nas
author: <noah>
---
## Configuración de Backups con Borg

Nuestro servidor de Unraid va a servir para guardar los backups de nuestras máquinas. En concreto vamos a guardar los backups de los equipos (portátiles y móviles) a través de Vorta, del VPS y del servidor Debian sanMi.  
Esta necesidad ha surgido por dos temas distintos:  
El primero es la necesidad de hacer backup de nuestros servidores secundarios. Podría instalar Duplicacy, que es el sistema que uso para hacer backup de unRaid en la cuenta de GoogleDrive pero visto lo bien que habla todo el mundo de borg vamos a probarlo una temporada.  
Segundo, los equipos de oficina estaban configurados con Gnome y usando la herramienta Deja-dup hacía los backups. El problema es que Deja-dup no funciona con KDE y me he visto obligado a emplear otra aplicación. Kbackup es una buena opción, pero no acaba de convencerme. Vorta me parece que es una opción excelente así que vamos a ello.  

### Instalación de borgserver
Buscamos en aplicaciones Unraid el servidor borg:

![borg-1.png](borg-1.png)

``` bash
Repository: nold360/borgserver:latest
Network Type: Bridge
SSH: 2222
# Ubicación de las claves ssh
sshkeys:/mnt/user/appdata/borg/sshkeys/
# Ubicación de los backups
backup: /mnt/user/Backups/borg
```

Pulsamos en aplicar y no arranca el contenedor. El motivo es que si no hemos añadido alguna clave ssh antes no arrancará.  
En la [web del proyecto](https://hub.docker.com/r/nold360/borgserver) de hub.docker.com lo explica:
**!IMPORTANT!: The container wouldn't start the SSH-Deamon until there is at least one ssh-keyfile in this directory!**  

``` bash
Copiamos nuestra clave pública a la siguiente ruta:
/mnt/user/appdata/borg/sshkeys/clients
```
En el momento de hacer este manual tengo dos claves públicas, la del pc portátil y la del servidor sanmi:
```bash
➜  clients pwd
/mnt/user/appdata/borg/sshkeys/clients
➜  clients ls   
envy_kde.pub  sanmi.pub
```

Ahora si reiniciamos el contenedor debería arrancar sin problemas

![borg-2.png](borg-2.png)

### Instalación de Vorta Backup en Fedora

En mi caso he instalado Vorta a través de Discover en formato flatpak.  
Iniciamos Vorta y comenzamos el procedimiento:  

Primero creamos nuestra clave ssh:

![borg-3.png](borg-3.png)

Una vez creada la clave ssh tenemos que copiarla a nuestro servidor borg a la siguiente ruta:
```bash
/mnt/user/appdata/borg/sshkeys/clients
```

Añadimos nuestro repositorio de Borg:

![borg-4.png](borg-4.png)

Es importante verificar correctamente los datos del repositorio. En mi caso:

```bash
PESTAÑA GENERAL:
Repository URL: ssh://borg@DIRECCION_IP_TAILSCALE:2222/./envy
Repository name: envy
Enter passphrase: XXXXXXXXX
PESTAÑA ADVANCED:
SSH Key: seleccionamos la clave ssh que hemos creado anteriormente
Encryption: Repokey-Blake2
```
El resto de opciones las dejamos por defecto.  
Planificamos la programación tanto de backups como de prune:

![borg-5.png](borg-5.png)

Con esto debería funcionar ya nuestro primer backup. Para restaurar es muy sencillo a través de Vorta. Procedemos a montar el backup que nos interese y realizamos la copia a nuestro sistema.  

### Instalación de borgmatic en servidor  
Aquí empieza el mondongo.  
Esta parte no ha sido sencilla hasta que di con la configuración adecuada.  





***
Fuentes:  
[https://hub.docker.com/r/nold360/borgserver](https://hub.docker.com/r/nold360/borgserver)  
