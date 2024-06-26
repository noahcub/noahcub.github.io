---
title: Securizando el proxy inverso swag
date: 2023-03-10 07:00:00 +0100
categories: [Nas]
tags: [software, config, unraid]     # TAG names should always be lowercase
img_path: /assets/pictures/Nas
author: <noah>
---

## Securizando SWAG  
Este post se podría considerar la segunda parte del post [Nextcloud y proxy inverso swag](/posts/Nextcloud-mariadb-swag/index.html).
En este punto intentaremos securizar en la medida de lo posible, y de mis conocimientos, nuestro proxy inverso SWAG para evitar cosas tan necesarias como por ejemplo ataques de fuerza bruta, conexiones desde IP en otros paises, etc.

He añadido notas a las que hago referencia en el post anterior pero no está demás recordarlas e incluso repasarlas por si se nos ha olvidado algo.

## CGNAT y Redirección puertos router
Es importante no estar en el CG-NAT de la operadora. En mi caso llevo tiempo con IP dinámica pero fuera del CG-NAT.  
Nuestro servidor nginx trabaja con los puertos 443 y 80. Como esas conexiones van a nuestro router, debemos redireccionar las peticiones que se hagan al puerto 443 y 80 hacia el contenedor de swag y los puertos que hayamos elegido para que funcione ese contenedor. 
Después Swag deriva las peticiones a cada servicio según la configuración. En mi caso, como unraid ya usa estos puertos he cambiado por estos otros, pero se puede poner cualquiera.

Por tanto, es imprescindible redireccionar los puertos del router a nuestro NAS:  
puerto 80 ROUTER ---> puerto xxx0 NAS  
puerto 443 ROUTER ---> puerto xxx3 NAS  

## Instalación de un Dashboard gráfico para swag  

Vamos a intentar securizar swag en la medida de lo posible.  
Lo primero que haremos es instalar un dashboard que nos permita ver la actividad del proxy inverso.  
Es importante recalcar que este dashboard solo funciona con subdominios y no con subdirectorios.

[SWAG Dashboard](https://github.com/linuxserver/docker-mods/tree/swag-dashboard)

Siguiendo las instrucciones de instalación es muy sencillo ponerlo en marcha en nuestro contenedor swag:  
Para ello modificamos el contenedor Swag de la siguiente manera:  
Añadimos la variable de entorno  

```bash 
DOCKER_MODS=linuxserver/mods:swag-dashboard
``` 
![swagmods](swag_mods.png)

Si tenemos más mods instalados hay que separarlos de la siguiente forma: 

```bash 
DOCKER_MODS=linuxserver/mods:swag-dashboard|linuxserver/mods:swag-mod2
```

Redireccionamos un puerto del unRaid al puerto 81 de nuestro contenedor swag para acceder al dashboard:

![swagports](swag_ports.png)

Por útlimo este es el resultado total:

![swagmodsok](swag_mods_ok.png)

Ya tenemos operativo el Dashboard. Para acceder ---> http://192.168.1.xxx:81

![Dashboard](dashboard.png)

## Securizando swag

### Capas de seguridad extra  
***Habilitar HSTS***  
HTTP Strict Transport Security (HSTS) es un estándar simple y ampliamente compatible para proteger a los visitantes al garantizar que sus navegadores siempre se conecten a un sitio web a través de HTTPS.  HSTS existe para eliminar la necesidad de la práctica común e insegura de redirigir a los usuarios de URL de http:// a https://.  

En nuestro caso, esta opción ya quedó habilitada ya que Nextcloud nos lo detectaba como un error de seguridad. Volvemos a revisarlo:  


```bash
cd /mnt/user/appdata/swag/nginx
cat ssl.conf 
```

Debe estar descomentala la línea siguiente:
```bash
# HSTS (ngx_http_headers_module is required) (63072000 seconds)
add_header Strict-Transport-Security "max-age=63072000" always;
# OCSP stapling
```

***X-Frame-Options***  
Con la cabecera X-Frame-Options y el parámetro SAMEORIGIN bloquearemos para que nuestra web no pueda cargar en ningún iframe excepto desde nuestro propio dominio.  

En nuestro caso, esta opción ya quedó habilitada ya que Nextcloud nos lo detectaba como un error de seguridad. Volvemos a revisarlo:  


```bash
cd /mnt/user/appdata/swag/nginx
nano ssl.conf 
```

```bash
#add_header Referrer-Policy "same-origin" always;
#add_header X-Content-Type-Options "nosniff" always;
add_header X-Frame-Options "SAMEORIGIN" always;
#add_header X-UA-Compatible "IE=Edge" always;
#add_header X-XSS-Protection "1; mode=block" always;
```
### Fail2ban
Fail2ban ya viene incluido en nustro servidor swag. Es una aplicación que se encarga de bloquear los ataques de fuerza bruta a nuestro servidor. 

Debemos localizar el fichero de logs de Nextcloud (en este caso, en otro caso sería otra aplicación):

En mi caso, Nextcloud guarda los logs en la ruta /mnt/user/appdata/nextcloud/config. Por defecto Nextcloud lo guarda aquí:
``` bash
Nextcloud-Files pwd
/mnt/user/Nextcloud-Files
Nextcloud-Files ls
admin                            index.html     
appdata_oclkxgc09392  files_external  nextcloud.log  

```
Nosotros vamos a modificar la ruta del fichero ***nextcloud.log*** para que se guarde en el pool de cache y no vaya a leer todo el tiempo al array:
``` bash
➜  nextcloud cd /mnt/user/appdata/nextcloud/config
➜  config ls
apache-pretty-urls.config.php  config.php         nextcloud.log             s3.config.php     upgrade-disable-web.config.php
apcu.config.php                config.php.bak     redis.config.php          smtp.config.php
apps.config.php                config.sample.php  reverse-proxy.config.php  swift.config.php

```
``` bash
➜  nano config.php
```
Y añadimos la siguiente línea:
``` bash
  'logfile' => '/var/www/html/config/nextcloud.log',
```
Que es la ubicación donde queremos guardar el nuevo fichero de logs, quedando así nuestro fichero de configuación:
``` bash
   [...]
  'mail_smtpport' => '465',
  'bulkupload.enabled' => false,
  'loglevel' => 0,
  'logfile' => '/var/www/html/config/nextcloud.log',
  'maintenance' => false,
  'maintenance_window_start' => 1,
);
```
Reiniciamos el contenedor de Nextcloud:
``` bash
➜  config docker restart Nextcloud                   
```

Una vez localizado procedemos a montar el fichero en modo solo lectura en el docker de swag, para que fail2ban pueda leer los intentos de conexión fallidos y proceder a su baneo:

![fail2ban](fail2ban.png)

Y actualizamos el contenedor swag.

Creamos el  fichero nextcloud.conf dentro de la carpeta fail2ban/filter.d del contenedor swag:

Fichero nextcloud.conf, añadimos el siguiente contenido:  

``` bash
[Definition]
failregex=^.*Login failed: '?.*'? \(Remote IP: '?<ADDR>'?\).*$
          ^.*\"remoteAddr\":\"<ADDR>\".*Trusted domain error.*$
ignoreregex =
```

Dentro de filter.d tenemos muchos ejemplos para filtros de distintos servicios.

Probamos si funciona la configuración aplicada:

``` bash
docker exec swag fail2ban-regex /nextcloud-logs/nextcloud.log /config/fail2ban/filter.d/nextcloud.conf

```
Debería salir algo similar a esto:

![fail2ban-2](fail2ban-2.png)

Por último tenemos que crear la jaula en el fichero general de fail2ban situado en fail2ban/jail.local. Añadimos esto al final del fichero:

``` bash
[nextcloud]
enabled  = true
filter   = nextcloud
port     = http,https
logpath  = /nextcloud-logs/nextcloud.log
action   =  iptables-allports[name=nextcloud]
```
Estos son los parámetros que podemos configurar en cada jaula:
    - enabled: Define if the jail is active or not  
    - filter: The filter name used for this jail, we will create it after  
    - port: Which standard ports it covers. It can include http, https, ssh, sftp and many others  
    - logpath: Path to the log file which is provided to the filter  
    - maxretry: Number of matches which triggers ban action on the IP  
    - bantime: Duration (in seconds) for IP to be banned for. Negative number for "permanent" ban  
Si no definimos algún parámetro, fail2ban aplicará los establecidos por defecto en la jaula [Default].  


Repetimos este proceso para cada uno de los servicio que expongamos a internet.

Si nos conectamos a una VPN podemos realizar la prueba haciendo unos logs erroneos:  
Como vemos en la imagen fail2ban ha bloqueado la ip de pruebas:
![fail2ban-3](fail2ban-3.png)

***Comandos útiles de fail2ban:***  
Ver estado de la jaula:  
``` bash
docker exec -it swag fail2ban-client status nextcloud
```

Listar jaulas activas:  
``` bash
docker exec -it swag fail2ban-client status
```

Listar IPs baneadas:
``` bash
docker exec -it swag fail2ban-client status <JAIL_NAME>
```

Banear IPs manualmente:
``` bash
docker exec -it swag fail2ban-client set <JAIL_NAME> banip <IP>
```

Desbanear una IP:  
``` bash
docker exec swag fail2ban-client unban <ip address>
```

### Habilitar notificaciones DISCORD de los baneos fail2ban

[F2B Discord Notification](https://github.com/linuxserver/docker-mods/tree/swag-f2bdiscord) - Docker mod which allows Fail2Ban Discord embeds

### Geoblock

Para configurar el bloqueo por geolocalización tenemos dos opciones:
[MAXMIND](https://github.com/linuxserver/docker-mods/tree/swag-maxmind) o [DBIP](https://github.com/linuxserver/docker-mods/tree/swag-dbip).  
En mi caso voy a usar la primera Maxmind.
Una vez registrados en MAXMIND tenemos que genera una licence key para que funcione:

![geoblock-1](geoblock-1.png)
![geoblock-2](geoblock-2.png)

Una vez creada la key modificamos el contenedor docker siguiendo las instrucciones del github de [MAXMIND](https://github.com/linuxserver/docker-mods/tree/swag-maxmind). Si tenemos varios MODS, como es mi caso, los separamos con "|":  

![geoblock-3](geoblock-3.png)

Añadimos la licence key como una nueva variable en el contenedor de swag:
![geoblock-4](geoblock-4.png)

Quedando definitivamente así:  
![geoblock-5](geoblock-5.png)

Dentro de la configuración del contenedor swag modificamos la configuración del fichero nginx.conf:

``` bash
cd /mnt/user/appdata/swag/nginx
nano nginx.conf
```
y añadimos lo siguiente en la sección http:

``` bash
include /config/nginx/maxmind.conf;
```
Luego configuramos nuestro fichero maxmind.conf según instrucciones de [https://virtualize.link/secure/](https://virtualize.link/secure/).  

Por último, añadimos la siguiente configuración en nuestros ficheros subdomain.conf de cada aplicación que usemos con swag:

``` bash
cd /mnt/user/appdata/swag/nginx/proxy-confs
nano nextcloud.subdomain.conf
```
``` bash
 server {
     listen 443 ssl;
     listen [::]:443 ssl;

     server_name some-app.*;
     include /config/nginx/ssl.conf;
     client_max_body_size 0;

     if ($lan-ip = yes) { set $geo-whitelist yes; }
     if ($geo-whitelist = no) { return 404; }

     location / {
``` 

Reiniciamos swag para que se apliquen los cambios...  

***   
Fuentes y enlaces de interés que ayudaran a complementar esta guía:  

[https://blog.thelazyfox.xyz/setup-swag-to-safely-expose-your-self-hosted-applications-to-the-internet/](https://blog.thelazyfox.xyz/setup-swag-to-safely-expose-your-self-hosted-applications-to-the-internet/)
[https://virtualize.link/secure/](https://virtualize.link/secure/)
