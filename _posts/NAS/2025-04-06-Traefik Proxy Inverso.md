---
title: Traefik Proxy Inverso - Cloudflare
date: 2025-04-06 07:00:00 +0100
categories: [Nas]
tags: [software, config, unraid]     # TAG names should always be lowercase
img_path: /assets/pictures/Nas
author: <noah>
---
## Instalación del proxy inverso Traefik con dominio personalizado en Cloudflare

Antes de comenzar con el artículo quiero dar las gracias a la gran comunidad de Unraid. Gracias a ellos he conseguido poner en marcha todo esto. En particular, agradecimientos especiales a los autores de los enlaces de interés que se citan al final de este post.  
  
Vamos a ello...
  
## CGNAT y Redirección puertos router  
  
Es importante no estar en el CG-NAT de la operadora. En mi caso llevo tiempo con IP dinámica pero fuera del CG-NAT.  
Nuestro servidor nginx trabaja con los puertos 443 y 80. Como esas conexiones van a nuestro router, debemos redireccionar las peticiones que se hagan al puerto 443 y 80 hacia el contenedor de traefik y los puertos que hayamos elegido para que funcione ese contenedor. 
Después Traefik deriva las peticiones a cada servicio según la configuración. En mi caso, como unraid ya usa estos puertos he cambiado por estos otros, pero se puede poner cualquiera.

Por tanto, es imprescindible redireccionar los puertos del router a nuestro NAS:  
puerto 80 ROUTER ---> puerto xxx0 NAS  
puerto 443 ROUTER ---> puerto xxx3 NAS  

## Servicio Cloudflare-DDNS
  
Este contenedor simplemente se encarga periodicamente de actualizar la dirección IP  a la que debe apuntar el dominio midominio.com. 

![swag-cloudflare](swag-cloudflare.png)

![swag-cloudflare-2](swag-cloudflare-2.png)

![swag-cloudflare-3](swag-cloudflare-3.png)

Vamos rellenando los datos según las imágenes:

Email: micorreo@correo.com
API Key: XXXXXXXXXXXXXXX - la creamos en nuestra cuenta de Cloudflare
Domain: midominio.com - registrado en Cloudflare

![swag-cloudflare-4.png](swag-cloudflare-4.png)

Iniciamos el contenedor y accediendo a los logs deberíamos observar como se está actualizando nuestra dirección IP:

```bash
No DNS update required for XXXXXXXXXXXXXX.com (XXX.XXX.XXX.XXX).
No DNS update required for XXXXXXXXXXXXXX.com (XXX.XXX.XXX.XXX).
```  

## Crear red personalizada docker
  
Debemos crear una red en docker a la que puedan conectarse los contenedores que vayan a usar swag. La red se llamará nginx.  
Nos conectamos por ssh al servidor y creamos la red docker:
```bash
docker network create cloud
docker network list
```  
Todos los contenedores creados a partir de ahora se conectarán a la custom network: cloud.


## Instalación de Traefik  
  
Antes de empezar, tenemos que accedera nuestro panel de cloudflare para crear el API Token que nos hará falta en la configuración del docker de Traefik.  
Lo preparamos con la configuración que tenemos en la siguiente imagen y después copiamos el token que nos da para añadirlo en el siguiete paso.  

![traefik-1](traefik-1.png)

Seleccionamos la aplicación Traefik del repositorio de [IBRACORP](https://docs.ibracorp.io/ibracorp).

![traefik-0](traefik-0.png)


Lo primero que destaca son los puertos de conexión y la url. 
Traefik emplea los puerto 80 y 443 para las conexiones entrantes, después deriva las peticiones a cada servicio según la configuración.

En este caso, es importante redireccionar los puertos del router al NAS:  
puerto 80 ROUTER ---> puerto xxx0 NAS  
puerto 443 ROUTER ---> puerto xxx3 NAS  

```bash
Network type: Custom: cloud  
Config folder: /mnt/user/appdata/traefik  
Docker Socket: /var/run/docker.sock  
Https port (Container Port: 443): xxxxx3  
Http port (Container Port: 80): xxxx3  
WebUI: xx443  
WebUI Port (Container Port: 8080): xxxx3  
URL: midominio.com  
Cloudflare API Token (Container Variable: CF_DNS_API_TOKEN): XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX  
Ahora vienen las etiquetas del contenedor:  

# Etiquetas traefik
Traefik Dashboard Subdomain (Container Label: traefik.http.routers.api.rule): Host(`traefik.MIDOMINIO.com`)  
Traefik API (Container Label: traefik.http.routers.api.service): api@internal  
Traefik Entrypoint (Container Label: traefik.http.routers.api.entryPoints): https  
Enable Traefik (Dashboard) (Container Label: traefik.enable): true  
```

Con esto el contenedor ya está configurado. Según le damos a aceptar **no arrancará** porque tenemos que crear los ficheros de configuración.   
  
Traefik necesita acceder al docker socket para monitorizar las etiquetas de los contenedores. Tenemos dos formas de hacerlo. IBRACORP lo explica muy bien. Vamos a instalar el **contendor dockersocket**. Se deja la configuración por defecto a excepción de la red, que seleccionamos **Custom: cloud**.  

Arrancamos el contenedor y para que traefik pueda acceder al docker socket por este medio tenemos que añadir una nueva variable:

![traefik-2](traefik-2.png)

También **es importante** realizar un cambio en el fichero traefik.yml. En el apartado **Docker provider**, debemos añadir el **endpoint: "tcp://dockersocket:2375"** según el recorte que tenemos a continuación:
  
``` bash
# Docker provider for connecting all apps that are inside of the docker network
  docker:
    watch: true
    network: my_custom_cloud    # Add Your Docker Network Name Here
    # Default host rule to containername.domain.example
    defaultRule: "Host(`{{ lower (trimPrefix `/` .Name )}}.YOURDOMAIN.COM`)"    # Replace with your domain
    # swarmModeRefreshSeconds: 15s #comment out or remove this line if using traefik v3
    exposedByDefault: false
    endpoint: "tcp://dockersocket:2375" # Uncomment if you are using docker socket proxy
  
```

## Ficheros de configuración  

Aunque ya nos hemos adelantado un poco con el fichero de configuración para el docker socket, vamos a ver en profundidad los ficheros necesarios para que funcione traefik.  
  
Archivo de configuración estática: traefik.yml   
Archivo de configuración dinámica: fileConfig.yml   
Archivo para guardar el certificado: acme.json  


``` bash
cd /mnt/user/appdata/traefik
touch acme.json
chmod 600 acme.json
```
En el github de UNRAIDES o en la web de IBRACORP tenemos modelos de los ficheros traefik.yml y fileConfig.yml.  
  
Configuración actual de mi **traefik.yml**:
``` bash
global:
  checkNewVersion: true
  sendAnonymousUsage: false

serversTransport:
  insecureSkipVerify: true

entryPoints:
  http:
    address: :80
    http:
      redirections:
        entryPoint:
          to: https
          scheme: https
  https:
    address: :443
    http:
      tls:
        certResolver: letsencrypt
        domains:
          - main: MIDOMINIO.com  #añadir tu dominio
            sans:
              - '*.MIDOMINIO.com'  #añadir tu dominio
      middlewares:
        - securityHeaders@file

providers:
  providersThrottleDuration: 2s

  # File provider for connecting things that are outside of docker / defining middleware
  file:
    filename: /etc/traefik/fileConfig.yml
    watch: true

  # Docker provider for connecting all apps that are inside of the docker network
  docker:
    watch: true
    network: cloud # red del proxyinverso
    defaultRule: "Host(`{{ index .Labels \"com.docker.compose.service\"}}.MIDOMINIO.com`)" #añadir tu dominio
    # swarmModeRefreshSeconds: 15s #comment out or remove this line if using traefik v3
    exposedByDefault: false
    endpoint: "tcp://dockersocket:2375" # Uncomment if you are using docker socket proxy

# Enable traefik ui
api:
  dashboard: true
  insecure: true

# Log level INFO|DEBUG|ERROR
log:
  level: INFO

# Use letsencrypt to generate ssl serficiates
certificatesResolvers:
  letsencrypt:
    acme:
      email: MICORREO@PERSONAL.com
      storage: /etc/traefik/acme.json
      dnsChallenge:
        provider: cloudflare
        resolvers:
          - "1.1.1.1:53"
          - "1.0.0.1:53"

```
  
Configuración actual de mi **fileConfig.yml**:
``` bash
http:
  routers:
    #añadir seguridad al dashboard
    dashboard:
      entryPoints:
        - https
      rule: Host(`traefik.MIDOMINIO.com`) && (PathPrefix(`/api`) || PathPrefix(`/dashboard`)) #añadir tu dominio
      service: api@internal
      middlewares:
        - auth
  ## MIDDLEWARES ##
  middlewares:
    # Auth
    #usuarios para autenticación
    #middleware para proteger el acceso al dashboard de traefik. No lo uso porque no tengo expuesto el dashboard
    #a internet. Solo tengo el acceso en mi red local
    auth:
      basicAuth:
        users:
          - "USER:PASS_HTPASSWD" #contraseña  con htpasswd 
# Security headers
    securityHeaders:
      headers:
        customResponseHeaders:
          # Mi nextcloud da un error de seguridad porque no tengo la cabecera X-Robots-Tag: "noindex,nofollow". Si se deja simplemente con "noindex,nofollow" no da el error pero es mejor añadir todas las demas.
          X-Robots-Tag: "noindex,nofollow,none,noarchive,nosnippet,notranslate,noimageindex"
          X-Forwarded-Proto: "https"
          server: ""
        customRequestHeaders:
          X-Forwarded-Proto: "https"
        sslProxyHeaders:
          X-Forwarded-Proto: "https"
        referrerPolicy: "same-origin"
        hostsProxyHeaders:
          - "X-Forwarded-Host"
        contentTypeNosniff: true
        browserXssFilter: true
        forceSTSHeader: true
        stsIncludeSubdomains: true
        stsSeconds: 63072000
        stsPreload: true

```
  
Una vez completada la configuración de los ficheros ya podemos acceder a nuestro dashboard de traefik:

![traefik-3](traefik-3.png)
  
## Exponer contenedores

Aquí está el asunto mas delicado. En los manuales que he consultado hay todo tipo de configuraciones y en algunos casos ha sido una locura hacerlo funcionar.  
A los contenedores docker les vamos a añadir unas etiquetas (**label**) que son las que indican a traefik que debe añadir el servicio en cuestión.  
En el [github de UnRAIDES](https://github.com/UnRAIDES/Scripts/tree/main/olivoa/traefik) lo explican muy bien:  
  
En la mayoría de los casos sólo es necesario usan las dos primeras etiquetas:
``` bash
    Name: Enable Traefik
    Key: traefik.enable
    Value: true
    Default value: true

    Name: https
    Key: traefik.http.routers.nombre-contenedor.entryPoints
    Value: https
    Default value: https

    # Esta la usaremos en caso de usar un subdominio distinto al nombre del contenedor docker
    Name: Subdominio
    Key: traefik.http.routers.nombre-contenedor.rule
    Value: Host(`subdominio.dominio.com`)

    # Contenedores que se conectan directamente a https
    Name: Esquema https
    Key: traefik.http.services.nombre-contenedor.loadbalancer.server.scheme
    Value: https
    Default value: https

    # Por si traefik no reconoce el puerto especificado en la plantilla
    Name: Ports
    Key: traefik.http.services.nombre-contenedor.loadbalancer.server.port
    Value: puerto personalizado
    Default: puerto personalizado
``` 

Voy a añadir algunos de los ejemplos que yo uso:

**Traefik** (son las etiquetas por defecto):
![traefik-4](traefik-4.png)
  
**Nextcloud**:
![traefik-5](traefik-5.png)
  
**Immich** (este coinicide literal con el manual de immich. Como lo tengo por docker-compose no hago captura):

``` bash
services:
  immich-server:
    container_name: immich_server
    image: ghcr.io/immich-app/immich-server:${IMMICH_VERSION:-release}
    volumes:
      - ${UPLOAD_LOCATION}:/usr/src/app/upload
      - /etc/localtime:/etc/localtime:ro
    env_file:
      - .env
    ports:
      - 2283:2283
    depends_on:
      - redis
      - database
    restart: always
    healthcheck:
      disable: false
    labels:
      traefik.enable: true
      traefik.http.routers.immich.rule: Host(`immich.MIDOMINIO.com`)
      traefik.http.routers.immich.entrypoints: https
      traefik.http.services.immich.loadbalancer.server.port: 2283
```
  
**Outline**:  
Este ha sido una locura. Para Outline he querido configurar autenticación OIDC a través de Authentik. Según la documentación de Authentik hay que hacerlo con un middleware pero no he sido capaz. Después de varios días perdiendo la cabeza y a punto de entrar en una espiral de locura encontré un [hilo de reddit](https://www.reddit.com/r/Authentik/comments/1dio0cw/authentik_traefik_paperlessngx_oauth/?tl=es-es) con un usuario que tenía el mismo error que yo:

![traefik-7](traefik-7.png)

En el hilo explicaba que tuvo que eliminar la etiqueta **traefik.http.routers.<appname>.middlewares: auth@files** para hacerlo funcionar. Y efectivamente, según borré esa etiqueta comenzó a funcionar sin problema.  
Estás son las etiquetas finales de mi docker Outline:

![traefik-6](traefik-6.png)
  
**Authentik**:
![traefik-8](traefik-8.png)

Vamos a poner las etiquetas de Authentik con más detalle:
``` bash
Container Label: traefik.enable: true
Container Label: traefik.http.routers.authentik.entryPoints: https
Container Label: traefik.http.routers.authentik.rule: Host(`authentik.MIDOMINIO.com`) || HostRegexp(`{subdomain:[A-Za-z0-9](?:[A-Za-z0-9\-]{0,61}[A-Za-z0-9])?}.MIDOMINIO.com`) && PathPrefix(`/outpost.goauthentik.io/`)
```
  
**Por último, se pueden añadir un montón de plugins a traefik. Pero eso queda para el siguiente manual...**

***   
Fuentes y enlaces de interés que ayudaran a complementar esta guía:  

[https://docs.ibracorp.io/traefik](https://docs.ibracorp.io/traefik)  
[https://diegoolivo.notion.site/Traefik-en-Unraid-45ed50c22bca4708adad442e1e92f2c4#c83b3526c9c5456ab747d3da6db57da4](https://diegoolivo.notion.site/Traefik-en-Unraid-45ed50c22bca4708adad442e1e92f2c4#c83b3526c9c5456ab747d3da6db57da4)  
[https://github.com/UnRAIDES/Scripts/tree/main/olivoa/traefik](https://github.com/UnRAIDES/Scripts/tree/main/olivoa/traefik)  
[https://www.reddit.com/r/Authentik/comments/1dio0cw/authentik_traefik_paperlessngx_oauth/?tl=es-es](https://www.reddit.com/r/Authentik/comments/1dio0cw/authentik_traefik_paperlessngx_oauth/?tl=es-es)  
[https://github.com/brokenscripts/authentik_traefik?tab=readme-ov-file](https://github.com/brokenscripts/authentik_traefik?tab=readme-ov-file)  
