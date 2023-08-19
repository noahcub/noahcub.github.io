---
title: Instalación y configuración de Fedora Server
date: 2023-08-18 07:00:00 +0100
categories: [Nas]
tags: [software, config, Fedora]     # TAG names should always be lowercase
img_path: /assets/pictures/Nas
author: <noah>
---

### Instalación y configuración básica del seridor

Descargamos desde la web de [Fedora](https://fedoraproject.org/server/) la imagen de netinstall y se instala como cualquier otro sistema. 

Una vez en funcionamiento accedemos al portal del Cockpit que se expone en el puerto 9090.
En mi caso: [https://192.168.1.22:9090/](https://192.168.1.22:9090/).

Lo primero es crear los usuarios que vayamos a necesitar desde Accounts. Al instalar creamos el primer usuario con permisos de administración y también configuramos la pass de root.
Nuevos usuarios:

![fedora-3](fedora-3.png)

Configuramos las actualizaciones automáticas del sistema, en mi caso solo actualizaciones de seguridad:

![fedora-4](fedora-4.png)

Configuramos las métricas del sistema que aportan información muy chula:

![fedora-5](fedora-5.png)

El sistema operativo ocupa muy poco espacio. En mi caso tengo un disco ssd de 480gb y el sistema ocupa tan solo 16gb. Para utilizar el resto accedemos a Storage:

![fedora-1](fedora-1.png)

Después hacemos click sobre "fedora/root" y creamos un nuevo volúmen lógico:

![fedora-2](fedora-2.png)

En este caso el nuevo volumen lo montamos sobre la carpeta /srv y ya tenemos disponible el resto del espacio del disco ssd.

A la carpeta **srv** debemos darle los permisos necesarios para los usuarios que accedan a ella

``` bash
drwxrwxrwx.   3 root root   18 Aug 19 13:13 srv

[user@localhost srv]$ ls -la
total 0
drwxrwxrwx.  3 root    root     18 Aug 19 13:13 .
dr-xr-xr-x. 18 root    root    255 Aug 18 23:26 ..
drwxr-xr-x.  4 user    user     33 Aug 19 13:16 CCTV
```

Instalación de htop, me resulta muy visual para ver procesos y rendimiento general del sistema:
``` bash
 sudo dnf install htop
 ```
Copiar nuestra clave ssh al servidor (despues podemos deshabilitar el acceso por password)

 ``` bash
 ssh-copy-id user@192.168.1.22
```

### Configuramos zerotier para acceder desde el exterior de la LAN:
``` bash
curl -s https://install.zerotier.com | sudo bash
```


### Instalación del servicio ftp para la grabación del sistema CCTV
``` bash
 sudo dnf upgrade -y
 sudo dnf install vsftpd
```
Configuración del fichero vsftpd.conf:

``` bash
 [root@www ~]# nano /etc/vsftpd/vsftpd.conf
# line 12: make sure value is [NO] (no anonymous)
anonymous_enable=NO

# line 83,84: uncomment (allow ascii mode)
ascii_upload_enable=YES
ascii_download_enable=YES

# line 101,102: uncomment (enable chroot)
chroot_local_user=YES
chroot_list_enable=YES
# line 104: uncomment (chroot list file)
chroot_list_file=/etc/vsftpd/chroot_list
# line 110: uncomment
ls_recurse_enable=YES
# line 115: change (if listening IPv4 only)
# if listning IPv4 and IPv6 both, specify [NO]
listen=YES
# line 124: change (if listening IPv6 only)
# if listning IPv4 and IPv6 both, specify [YES]
listen_ipv6=NO
# add to the end
# specify root directory (if don't specify, users' home directory become FTP home directory)
local_root=public_html
# use local time
use_localtime=YES
# turn off for seccomp filter (if cannot login, add this line)
seccomp_sandbox=NO
```

``` bash
 [root@www ~]# vi /etc/vsftpd/chroot_list
# add users you allow to move over their home directory
fedora
user
cctv
```
``` bash
[root@www ~]# systemctl enable --now vsftpd 
```

Permitimos en el firewall el servicio FTP:
``` bash
 [root@www ~]# firewall-cmd --add-service=ftp --permanent
success
[root@www ~]# firewall-cmd --reload
success 
```
Damos permiso en SELinux:
``` bash
[root@www ~]# setsebool -P ftpd_full_access on 
```

Probando el acceso desde otro dispositivo con un usuario del sistema debería funcionar, accediendo a su carpeta de usuario home.   
En mi caso como el volumen de almacenamiento de datos se encuentra en **/srv**, he creado un enlace simbólico en la carpeta de usuario para grabar los datos de las cámaras en ese volumen:  
**OJO con los permisos de usuario de las carpetas**
``` bash
mkdir /srv/CCTV
cd /home/user/
ln -s /srv/CCTV/ CCTV
```
En su momento estuve haciendo pruebas para que el usuario pudiera acceder por ftp directamente a la carpeta **/srv/CCTV** pero no había manera de acceder y como no tenía ganas de buscar información o perder el tiempo decidí arreglarlo con un enlace simbólico.





***   
Fuentes y enlaces de interés que ayudaran a complementar esta guía:  

[https://www.server-world.info/en/note?os=Fedora_32&p=ftp&f=1](https://www.server-world.info/en/note?os=Fedora_32&p=ftp&f=1)
