---
title: Nextcloud y proxy inverso swag
date: 2023-02-27 07:00:00 +0100
categories: [Nas]
tags: [software, config, unraid]     # TAG names should always be lowercase
img_path: /assets/pictures/Nas
author: <noah>
---
## Instalación de Nextcloud con proxy inverso Swag
Actualmente tengo un NAS con sistema operativo unRAID. Vamos a explicar la forma de instalar Nextcloud con Mariadb. Para acceder desde internet usaremos el proxy inverso Swag. Este proxy nos permite acceder a otros servicios, pero seo lo dejamos para otro post.

## CGNAT y Redirección puertos router
Es importante no estar en el CG-NAT de la operadora. En mi caso llevo tiempo con IP dinámica pero fuera del CG-NAT.  
Nuestro servidor nginx trabaja con los puertos 443 y 80. Como esas conexiones van a nuestro router, debemos redireccionar las peticiones que se hagan al puerto 443 y 80 hacia el contenedor de swag y los puertos que hayamos elegido para que funcione ese contenedor. 
Después Swag deriva las peticiones a cada servicio según la configuración. En mi caso, como unraid ya usa estos puertos he cambiado por estos otros, pero se puede poner cualquiera.

Por tanto, es imprescindible redireccionar los puertos del router a nuestro NAS:  
puerto 80 ROUTER ---> puerto xxx0 NAS  
puerto 443 ROUTER ---> puerto xxx3 NAS  

## Servicio duckdns

Este manual es una actualización y creo que este punto no es necesario, pero lo dejo por si acaso.  
Antes lo tenía funcionando porque usaba nginx-proxy-manager, pero con swag creo que no hace falta.

![duckdns](duckdns.png)

Este contenedor simplemente se encarga periodicamente de actualizar la dirección IP  a la que debe apuntar el dominio ssssssssss.duckdns.org. Como esto ya se configura en swag, creo que no es necesario. 

## Crear red personalizada docker

Debemos crear una red en docker a la que puedan conectarse los contenedores que vayan a usar swag. La red se llamará nginx.  
Nos conectamos por ssh al servidor y creamos la red docker:
```bash
docker network create nginx
docker network list
```  

Todos los contenedores creados a partir de ahora se conectarán a la custom network: nginx.


## Instalación de Swag

![swag-0](swag-0.png)

Lo primero que destaca son los puertos de conexión y la url. 
Swag emplea los puerto 80 y 443 para las conexiones entrantes, después deriva las peticiones a cada servicio según la configuración. En mi caso, como unraid ya usa estos puertos he cambiado por estos otros, pero se puede poner cualquiera.

En este caso, es importante redireccionar los puertos del router al NAS:  
puerto 80 ROUTER ---> puerto xxx0 NAS  
puerto 443 ROUTER ---> puerto xxx3 NAS  

Respecto a la url, yo uso duckdns para mi dominio gratuito, por tanto sería:  
dominio.duckdns.org  

![swag-1](swag-1.png)

***Importante que el tipo de red debe ser personalizada: la red que hemos creado para usar con nginx.***

Continuamos con la configuración añadiendo el correo electrónico para las notificaciones de caducidad del certificado y en la variable SUBDOMAINS empleamos el comodin wildcard que nos permite usar los subdominios que queramos. También podríamos poner separados por comas cada subdominio: nextcloud, plex, o lo que usemos.

![swag-2](swag-2.png)


El resto sigui todo por defecto:

![swag-3](swag-3.png)

Cuando arrancamos swag nos genera un error porque falta una variable de duckdns, que esel token de acceso:  

![swag-4](swag-4error.png)

Simplemente acudimos a la ruta indicada y añadimos los datos que nos pide:


``` bash
cd /mnt/user/appdata/swag/dns-conf

nano duckdns.ini
# Instructions: https://github.com/infinityofspace/certbot_dns_duckdns#credenti>
# Replace with your API token from your duckdns account.
dns_duckdns_token=NUESTRO TOKEN DE DUCKDNS

docker restart swag
```
Una vez reiniciado el contenedor debería funcionar correctamente:  

![swag-5](swag-5ok.png)

Realizando una prueba de acceso desde el navegador debería funcionar nuestro servidor nginx:

![swag-6](swag-6test.png)


## Instalación de mariadb

En mariadb se almacenará la base de datos de nextcloud.  

![mariadb0](mariadb-0.png)

Acordarse de añadir la red nginx.

![mariadb1](mariadb-1.png)

![mariadb2](mariadb-2.png)

Al decirle que genere la password de root aleatoria, debemos acceder a los logs del contenedor para tomar nota y guardarla para posibles inconvenientes. 
Marcamos los nombres de las variables de la base de datos que vamos a usar con nextcloud:  
Data base user  
Data base name  
Data base password  

Podemos comprobar el correcto funcionamiento de la base de datos a través del contenedor adminer (muy sencillo de usar e instalar si no se tiene conocimientos de mysql):

![adminer](adminer.png)

## Instalación de Nextcloud

Comenzamos creando una nueva carpeta compartida donde almacenaremos los ficheros e Nexcloud de los usuarios:

![Share](share-unraid.png)

Después instalamos el contenedor Nextcloud-official.  

![nextcloud-0](nextcloud-0.png)

Elegimos las opciones por defecto, salvo la ubicación de los ficheros de usuario, que ponemos la carpeta compartida que hemos creado.  
Acordarse de añadir la red nginx.

***Importante, según notas de configuración de swag, para que funcione correctamente el contenedor debe llamarse nextcloud***

![nextcloud-1](nextcloud-1.png)

![nextcloud-2](nextcloud-2.png)

Revisamos los logs del contenedor y una vez esté listo accedemos localmente para crear nuestro usuario administrador y configurar la base de datos:  
***Importante, con firefox me da fallo. He tenido que hacer esta configuración con Brave*** 

![nextcloud-3](nextcloud-3.png)

Nextcloud nos preguntará si queremos instalar las aplicaciones recomendadas (en mi caso no):

![nextcloud-4](nextcloud-4.png)

## Configuración proxy inverso swag y nextcloud

``` bash
cd /mnt/user/appdata/swag/nginx/proxy-confs
cp nextcloud.subdomain.conf.sample nextcloud.subdomain.conf
```
Si hacemos un cat al fichero nextcloud.subdomain.conf, nos informan de los pasos a seguir para hacer funcionar el servicio nextcloud con swag:  

``` bash
cat nextcloud.subdomain.conf                               
## Version 2023/02/05
# make sure that your nextcloud container is named nextcloud
# make sure that your dns has a cname set for nextcloud
# assuming this container is called "swag", edit your nextcloud container's config
# located at /config/www/nextcloud/config/config.php and add the following lines before the ");":
#  'trusted_proxies' => ['swag'],
#  'overwrite.cli.url' => 'https://nextcloud.example.com/',
#  'overwritehost' => 'nextcloud.example.com',
#  'overwriteprotocol' => 'https',
#
# Also don't forget to add your domain name to the trusted domains array. It should look somewhat like this:
#  array (
#    0 => '192.168.0.1:444', # This line may look different on your setup, don't modify it.
#    1 => 'nextcloud.example.com',
#  ),
``` 

Pues nos vamos al fichero de configuración de nextcloud y lo modificamos: 

``` bash
cd /mnt/user/appdata/nextcloud/config
cp config.php config.php.bak
nano config.php
```
Quedando el fichero finalmente así el fichero de Nextcloud config.php:  
``` bash
<?php
$CONFIG = array (
[.....................]
  'trusted_domains' => 
  array (
    0 => '192.168.1.xxxx:8666',
    1 => 'nextcloud.xxxxxxxxxxxxxxxxxxxxxxxxxx.duckdns.org',
  ),
  'datadirectory' => '/var/www/html/data',
  'dbtype' => 'mysql',
  'version' => '25.0.4.1',
  'trusted_proxies' => ['swag'],
  'overwrite.cli.url' => 'https://nextcloud.xxxxxxxxxxxxxxxxxx.duckdns.org/',
  'overwritehost' => 'nextcloud.xxxxxxxxxxxxxxxxxxxxxxxxx.duckdns.org',
  'overwriteprotocol' => 'https',
  'dbname' => 'nextcloud',
  'dbhost' => '192.168.1.xxxx:3306',
  'dbport' => '',
  'dbtableprefix' => 'oc_',
  'mysql.utf8mb4' => true,
  'dbuser' => 'nextcloud_user',
  'dbpassword' => 'xxxxxxxxxxxxxxxxxxxxxxxxxxx',
  'installed' => true,
);
```

Al instalar el contenedor de nextcloud dice que el proxy inverso debe comunicarse con Nextcloud por http:  

![config-1](config-1.png)

Para que funcione he dejado el fichero de configuración de swag nextcloud.subdomain.conf de la siguiente forma:  

``` bash
cd /mnt/user/appdata/swag/nginx/proxy-confs
nano nextcloud.subdomain.conf
```

``` bash
server {
    listen 443 ssl;
    listen [::]:443 ssl;

    server_name nextcloud.*;

    include /config/nginx/ssl.conf;

    client_max_body_size 0;

    location / {
        include /config/nginx/proxy.conf;
        include /config/nginx/resolver.conf;
        set $upstream_app nextcloud;
        set $upstream_port 80;
        set $upstream_proto http;
        proxy_pass $upstream_proto://$upstream_app:$upstream_port;

        proxy_hide_header X-Frame-Options;
        proxy_max_temp_file_size 2048m;
    }
}

```

Reiniciamos los dos contenedores:

``` bash
docker restart swag
docker restart nextcloud
```
Y probamos el acceso, esta vez desde el dominio personalizado:

![config-2](config-2.png)

Ya debería funcionar.  

## Corrección de errores de seguridad 

Accedemos como administrador y en el apartado Security & setup warnings vemos varios errores que tenemos que corregir para hacer más seguro nuestro nextcloud:

![postconfig-1](postconfig-1.png)

El primer error lo solucionamos descomentando la siguiente línea en el fichero ssl.conf de nginx:

``` bash
cd /mnt/user/appdata/swag/nginx
nano ssl.conf 
```

``` bash
#add_header Referrer-Policy "same-origin" always;
#add_header X-Content-Type-Options "nosniff" always;
add_header X-Frame-Options "SAMEORIGIN" always;
#add_header X-UA-Compatible "IE=Edge" always;
#add_header X-XSS-Protection "1; mode=block" always;
```

El segundo error se corrige descomentando la siguiente línea del mismo fichero ssl.conf:

``` bash
# HSTS (ngx_http_headers_module is required) (63072000 seconds)
add_header Strict-Transport-Security "max-age=63072000" always;
# OCSP stapling
```
El error del correo electrónico se soluciona configurando un correo que permita enviar correos para notificaciones:  

![postconfig-2](postconfig-2.png)

Para el último error de la región del teléfono se soluciona si añadimos el siguiente texto al fichero config.php de nextcloud:

``` bash
  'installed' => true,
  'default_phone_region' => 'ES'
```

Reiniciamos los dos contenedores:
``` bash
docker restart swag
swag
docker restart nextcloud
nextcloud
``` 
Verificamos que todo está correcto:

![postconfig-3](postconfig-3.png)

Realizamos el escaneo de seguridad externo de nextcloud:

![postconfig-4](postconfig-4.png)

Y vemos el resultado final:

![postconfig-5](postconfig-5.png)

Por útlimo habría que crear los usuarios e instalar las aplicaciones más interesantes. Es importante habilitar 2FA en los usuarios. Para ellos activamos la siguiente aplicación:

![totp](totp.png)

a debería funcionar.

## Security & setup warnings cron  job error
En el apartado de setup de Nextcloud nos aparecen los errores del servidor.  
Uno de los errores es el siguiente: Last background job execution ran 10 hours ago. Something seems wrong.  
En los foros de Unraid encontré una solución:
![unraid-cron](unraid-cron.png)

Creamos un script en User Scripts de Unraid con el siguiente texto:

```bash
echo "START SCANNING FOR NEXTCLOUD FILES&FOLDERS"
docker exec --user 99 Nextcloud php occ files:scan --all &
docker exec --user 99 Nextcloud php -f /var/www/html/cron.php
```
y programamos que se ejecute cada hora. Si ejecutamos en consola como prueba debería funcionar sin problemas:  

![unraid-cron-2](unraid-cron-2.png)

## Security & setup warnings cron  job error
Otro de los errores que obtenemos es el siguiente:  
This instance is missing some recommended PHP modules. For improved performance and better compatibility it is highly recommended to install them: bz2.

Buscando en los foros de Nextcloud se soluciona de la siguiente forma:  
Accedemos al contenedor con usuario administrador:
```bash
docker exec -u 0 -ti Nextcloud /bin/bash
```
y ejecutamos el siguiente comando:
```bash
apt-get update && apt-get install -y libbz2-dev && docker-php-ext-install bz2
```
Por motivo que desconozco la primera vez que lo ejecutamos Nextcloud sigue dando el mismo error. Sin embargo, según el [mensaje del usuario jn1000](https://help.nextcloud.com/t/docker-image-setup-warning-missing-bz2-after-update-to-nc-28-0-0/176605/6) cuando se hace por segunda vez ya queda corregido el error.  

## Errores en la sincronización  
Durante la sincronización Nextcloud daba el siguiente error:

``` bash
network error 99 nextcloud
```
La sincronización iba como a golpes y no acababa de funcionar correctamente.  

Buscando soluciones encontré esto por github:

``` bash
I have been hitting “Network Error 99” and the Desktop client freezing up since upgrading to NextCloud 25. I finally found a fix discussed in GitHub issue 5094 150, in summary this is what fixed it for me:

    On the client, disable any upload speed limits
    On the server, edit config.php to add the line 'bulkupload.enabled' => false,

From the discussions on GitHub, it sounds like the upload speed limit and bulk upload features are buggy in the latest server/desktop releases.
```
Editamos el fichero config.php de nextcloud y añadimos la línea 
``` bash
'bulkupload.enabled' => false,
```
Parece que se solucionó y comenzó a funcionar correctamente la sincronización.

## Servidor en mantenimiento - maintenance mode: true 

En alguna ocasión cuando hay disponible una actualización de nextcloud, el servidor entra en modo mantenimiento (maintenance mode: true) y no inicia. 
Por lo que he leido significa que tiene pendiente algun tipo de actualización. Para completar la actualización debemos hacer lo siguiente:

Nos conectamos por ssh al NAS y verificamos que el modo de mantenimiento está activado:

Editamos el fichero config.php de nextcloud: 
``` bash
nano /mnt/user/appdata/nextcloud/config/config.php
```
Buscamos la línea   'maintenance' => true,
``` bash
'maintenance' => true,
```

Ahora podemos iniciar la actualización:  
Abrimos una consola en el docker de nextcloud y ejecutamos:
``` bash
$ php occ upgrade
Nextcloud or one of the apps require upgrade - only a limited number of commands are available
You may use your browser or the occ upgrade command to do the upgrade
Setting log level to debug
Updating database schema
Updated database
Disabled incompatible app: recognize
Update app 
[....]
Starting code integrity check...
Finished code integrity check
Update successful
Maintenance mode is kept active
Resetting log level
```

Quitamos el modo mantenimiento:
``` bash
'maintenance' => false,
```

Reiniciamos el contenedor y ya debería funcionar sin problemas.  

***   
Fuentes y enlaces de interés que ayudaran a complementar esta guía:  

Instalación Nextcloud:  
[https://telegra.ph/Como-configurar-SWAG-como-proxy-inverso-en-Unraid-y-cualquier-otro-NAS-usando-docker-05-09](https://telegra.ph/Como-configurar-SWAG-como-proxy-inverso-en-Unraid-y-cualquier-otro-NAS-usando-docker-05-09)  
[https://forum.openmediavault.org/index.php?thread/28216-how-to-nextcloud-with-swag-letsencrypt-using-omv-and-docker-compose/](https://forum.openmediavault.org/index.php?thread/28216-how-to-nextcloud-with-swag-letsencrypt-using-omv-and-docker-compose/)  
[https://docs.nextcloud.com/server/latest/admin_manual/maintenance/backup.html](https://docs.nextcloud.com/server/latest/admin_manual/maintenance/backup.html)
[https://docs.nextcloud.com/server/latest/admin_manual/maintenance/manual_upgrade.html](https://docs.nextcloud.com/server/latest/admin_manual/maintenance/manual_upgrade.html)
[https://forums.unraid.net/topic/88504-support-knex666-nextcloud/page/21/](https://forums.unraid.net/topic/88504-support-knex666-nextcloud/page/21/)


