---
title: Traefik v3 y Crowdsec
description: Configuración de traefik v3 con crowdsec
date: 2026-01-23 07:00:00 +01:00
categories: [Nas]
tags: [software, config, unraid]     # TAG names should always be lowercase
img_path: /assets/pictures/Nas
author: <noah>
---

## Traefik y crowdsec
  
Hasta hace poco tenía en funcionamiento traefik con crowdsec usando las plantillas de unraid. Hice varios manuales sobre la instalación pero por motivos que desconozco crowdsec no parseaba las líneas de logs de traefik y por tanto no hacía absolutamente nada.  
He decicido hacer una instalación limpia desde 0 y aquí va.

### Preparación del entorno
Lo primero que vamos a hacer es preparar nuestro entorno de directorios:  

Directorios de traefik y crowdsec:
``` bash
mkdir -p /mnt/user/appdata/traefik/{letsencrypt,log}
mkdir -p /mnt/user/appdata/crowdsec/data
```

Ficheros de configuración y logs de traefik:
``` bash
touch /mnt/user/appdata/traefik/letsencrypt/acme.json
chmod 600 /mnt/user/appdata/traefik/letsencrypt/acme.json

touch /mnt/user/appdata/traefik/traefik.yml
touch /mnt/user/appdata/traefik/dynamic_conf.yml

touch /mnt/user/appdata/traefik/log/access.log
chmod 644 /mnt/user/appdata/traefik/log/access.log
```

Fichero de configuración de crowdsec. Dentro de la carpeta de configuración de crowdsec creamos el fichero acquis.yaml:
``` bash
touch /mnt/user/appdata/crowdsec/config/acquis.yaml
```

### Docker-compose
Sigo usando Unraid. Nos vamos a la pestaña compose y creamos nuestro stack de docker:

**docker-compose.yml**
``` bash
services:
  traefik:
    image: traefik:v3.0
    container_name: traefik
    restart: unless-stopped
    security_opt:
      - no-new-privileges:true
    ports:
      - "1880:80"
      - "18443:443"
    environment:
      - CF_DNS_API_TOKEN=${CF_DNS_API_TOKEN} # API TOKEN DE CLOUDFLARE
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock:ro
      - /mnt/user/appdata/traefik/letsencrypt/acme.json:/letsencrypt/acme.json
      - /mnt/user/appdata/traefik/traefik.yml:/etc/traefik/traefik.yml:ro
      - /mnt/user/appdata/traefik/dynamic_conf.yml:/etc/traefik/dynamic_conf.yml:ro
      - /mnt/user/appdata/traefik/log:/var/log/traefik # Volumen para persistencia de logs
    networks:
      - cloud
      
  crowdsec:
    image: crowdsecurity/crowdsec
    container_name: crowdsec
    restart: unless-stopped
    environment:
      - COLLECTIONS=crowdsecurity/traefik crowdsecurity/http-cve
    volumes:
      - /mnt/user/appdata/traefik/log:/var/log/traefik:ro   # comparte los logs
      - /mnt/user/appdata/crowdsec/data:/var/lib/crowdsec/data
      - /mnt/user/appdata/crowdsec/config:/etc/crowdsec
    networks:
      - cloud

networks:
  cloud:
    external: true
```

**Fichero .env**
``` bash
CF_DNS_API_TOKEN=MI_API_KEY_CLOUDFLARE. # Se obtiene en el panel de cloudflare
CROWDSEC_BOUNCER_API_KEY= xxxxxxxx # Con la última versión del docker-compose se puede borrar
DOMAIN_NAME=mi_dominio.com  # Con la última versión del docker-compose se puede borrar
EMAIL=mi_correo@hotmail.com # Con la última versión del docker-compose se puede borrar
```

**Nota Importante: ES IMPRESCINDIBLE QUE LOS SERVICIOS QUE USEMOS CON TRAEFIK ESTÉN EN LA MISMA RED DOCKER. En mi caso se llama cloud**  

### Ficheros de configuración

**Traefik - traefik.yml**
``` bash
api:
  dashboard: false # Cuando necesito revisar el dashboard lo cambio a true y reinicio el contenedor. Mientras no sea necesario lo dejo en false.

providers:
  docker:
    exposedByDefault: false
    network: cloud # Debe coincidir con el nombre de tu red externa en docker-compose
  file:
    fileName: /etc/traefik/dynamic_conf.yml
    watch: true

entryPoints:
  web:
    address: ":80"
    http:
      redirections:
        entryPoint:
          to: websecure
          scheme: https
  websecure:
    address: ":443"
    transport:
      respondingTimeouts:
        idleTimeout: 3600

# 1. HABILITAR LOGS (Vital para que CrowdSec detecte ataques)
accessLog:
  filePath: "/var/log/traefik/access.log"
  format: json
  bufferingSize: 100
  fields:
    headers:
      defaultMode: keep # Mantiene headers para que CrowdSec vea IPs reales

# 2. PLUGINS (CrowdSec Bouncer para Traefik v3)
#
# AQUI AÑADIREMOS EL PLUGIN DE CROWDSEC
#

# 3. CONFIGURACIÓN DE CERTIFICADOS (Cloudflare DNS Challenge)
certificatesResolvers:
  cloudflare:
    acme:
      email: mi_correo@hotmail.com
      storage: /letsencrypt/acme.json
      dnsChallenge:
        provider: cloudflare
        resolvers:
          - "1.1.1.1:53"
          - "8.8.8.8:53"
``` 

**Traefik - dynamic_conf.yml**
``` bash
http:
  routers:
    dashboard:
      rule: "Host(`traefik.mi_dominio.com`)"
      service: api@internal
      entryPoints:
        - websecure
      tls:
        certResolver: cloudflare
      middlewares:
        - auth
        - security-headers

  middlewares:
    auth:
      basicAuth:
        # Generar con: echo $(htpasswd -nB usuario) | sed -e s/\\$/\\$\\$/g
        users:
          - "noah:$mi_pass_hasheada"

    security-headers:
      headers:
        stsSeconds: 15552000
        stsIncludeSubdomains: true
        stsPreload: true
        forceSTSHeader: true
        frameDeny: true # Previene Clickjacking
        contentTypeNosniff: true # Previene sniffing de MIME
        browserXssFilter: true
        referrerPolicy: "same-origin"
        # Ajuste para Nextcloud:
        customFrameOptionsValue: "SAMEORIGIN"
    #
    # AQUI AÑADIREMOS EL MIDDLEWARE DE CROWDSEC
    #
```

**Crowdsec - acquis.yaml**
``` bash
---
filenames:
  - /var/log/traefik/access.log
poll_without_inotify: true
labels:
  type: traefik
```

### Integración del bouncer en traefik   

Iniciamos nuestro compose y no debería lanzar errores.  
Es momento de crear nuestra API key del bouncer crowdsec:  

``` bash
---
docker exec -it crowdsec cscli bouncers add traefik-bouncer
# te devolverá una API key, guárdala
```

Verificamos que nuestras colecciones estén actualizadas:

``` bash
docker exec -it crowdsec cscli hub update
docker exec -it crowdsec cscli hub upgrade
docker restart crowdsec
```

Modificamos nuestros ficheros traefik.yml y dynamic_conf.yml para añadir lo siguente:  

**Traefik - traefik.yml**
``` bash
experimental:
  plugins:
    crowdsec-bouncer:
      moduleName: github.com/maxlerebourg/crowdsec-bouncer-traefik-plugin
      version: v1.3.5
``` 

**Traefik - dynamic_conf.yml**
``` bash
  middlewares:
    [...AQUI TENEMOS NUESTROS OTROS MIDDLEWARES......]
    crowdsec-bouncer:
      plugin:
        crowdsec-bouncer:
          Enabled: true
          CrowdsecMode: live          # o streaming si prefieres
          CrowdsecLapiUrl: "http://crowdsec:8080"
          CrowdsecLapiKey: "ESTA KEY SE GENERA MAS ADELANTE"
          ForwardedHeadersCustomName: "X-Forwarded-For"
          # Obtenemos la lista oficial de IPs con el siguiente comando:
          # curl https://www.cloudflare.com/ips-v4 -o cloudflare-ips-v4.txt
          ForwardedHeadersTrustedIps:
            - "103.21.244.0/22"
            - "103.22.200.0/22"
            - "103.31.4.0/22"
            - "104.16.0.0/13"
            - "104.24.0.0/14"
            - "108.162.192.0/18"
            - "131.0.72.0/22"
            - "141.101.64.0/18"
            - "162.158.0.0/15"
            - "172.64.0.0/13"
            - "173.245.48.0/20"
            - "188.114.96.0/20"
            - "190.93.240.0/20"
            - "197.234.240.0/22"
            - "198.41.128.0/17"
```

**Ficheros completos:**  

**Traefik - traefik.yml**  
``` bash
api:
  dashboard: false # Cuando necesito revisar el dashboard lo cambio a true y reinicio el contenedor. Mientras no sea necesario lo dejo en false.

providers:
  docker:
    exposedByDefault: false
    network: cloud # Debe coincidir con el nombre de tu red externa en docker-compose
  file:
    fileName: /etc/traefik/dynamic_conf.yml
    watch: true

entryPoints:
  web:
    address: ":80"
    http:
      redirections:
        entryPoint:
          to: websecure
          scheme: https
  websecure:
    address: ":443"
    transport:
      respondingTimeouts:
        idleTimeout: 3600

# 1. HABILITAR LOGS (Vital para que CrowdSec detecte ataques)
accessLog:
  filePath: "/var/log/traefik/access.log"
  format: json
  bufferingSize: 100
  fields:
    headers:
      defaultMode: keep # Mantiene headers para que CrowdSec vea IPs reales

# 2. PLUGINS (CrowdSec Bouncer para Traefik v3)
experimental:
  plugins:
    crowdsec-bouncer:
      moduleName: github.com/maxlerebourg/crowdsec-bouncer-traefik-plugin
      version: v1.3.5

# 3. CONFIGURACIÓN DE CERTIFICADOS (Cloudflare DNS Challenge)
certificatesResolvers:
  cloudflare:
    acme:
      email: mi_correo@hotmail.com
      storage: /letsencrypt/acme.json
      dnsChallenge:
        provider: cloudflare
        resolvers:
          - "1.1.1.1:53"
          - "8.8.8.8:53"
``` 

**Traefik - dynamic_conf.yml**
``` bash
http:
  routers:
    dashboard:
      rule: "Host(`traefik.mi_dominio.com`)"
      service: api@internal
      entryPoints:
        - websecure
      tls:
        certResolver: cloudflare
      middlewares:
        - auth
        - security-headers
        - crowdsec-bouncer # Protegemos el dashboard con crowdsec


  middlewares:
    auth:
      basicAuth:
        # Generar con: echo $(htpasswd -nB usuario) | sed -e s/\\$/\\$\\$/g
        users:
          - "noah:$mi_pass_hasheada"

    security-headers:
      headers:
        stsSeconds: 15552000
        stsIncludeSubdomains: true
        stsPreload: true
        forceSTSHeader: true
        frameDeny: true # Previene Clickjacking
        contentTypeNosniff: true # Previene sniffing de MIME
        browserXssFilter: true
        referrerPolicy: "same-origin"
        # Ajuste para Nextcloud:
        customFrameOptionsValue: "SAMEORIGIN"
    
    crowdsec-bouncer:
      plugin:
        crowdsec-bouncer:
          Enabled: true
          CrowdsecMode: live          # o streaming si prefieres
          CrowdsecLapiUrl: "http://crowdsec:8080"
          CrowdsecLapiKey: "zEtyY2gJsQgQca03vEvWWwcowSG8f9yJz84nF95qZq4"
          ForwardedHeadersCustomName: "X-Forwarded-For"
          ForwardedHeadersTrustedIps:
            - "103.21.244.0/22"
            - "103.22.200.0/22"
            - "103.31.4.0/22"
            - "104.16.0.0/13"
            - "104.24.0.0/14"
            - "108.162.192.0/18"
            - "131.0.72.0/22"
            - "141.101.64.0/18"
            - "162.158.0.0/15"
            - "172.64.0.0/13"
            - "173.245.48.0/20"
            - "188.114.96.0/20"
            - "190.93.240.0/20"
            - "197.234.240.0/22"
            - "198.41.128.0/17"
```

### Obtener las IPs confiables de cloudflare  

Para configurar ForwardedHeadersTrustedIps con precisión en el middleware de CrowdSec, se usan las IPs de Cloudflare. Esto evita que IPs falsas se usen para evadir bloqueos.  
​
Lista oficial de IPs de Cloudflare.  

Cloudflare publica sus rangos IPv4/IPv6 en dos archivos JSON. Los podemos obetner así
```bash
# Desde la terminal de Unraid (donde corre Docker)
curl https://www.cloudflare.com/ips-v4 -o cloudflare-ips-v4.txt
curl https://www.cloudflare.com/ips-v6 -o cloudflare-ips-v6.txt
```
Las IPs que obtenemos las añadimos al fichero dynamic_conf.yml, apartado **ForwardedHeadersTrustedIps** del middleware crowdsec-bouncer.  


### Puesta en marcha y comprobación  

Con todo esto reiniciamos el stack de docker y debería arrancar funcionando sin errores. Revisaremos los logs para verificar fallos.
```bash
docker compose down
docker compose up -d
# En Unraid lo hacemos de forma gráfica
```

```bash
docker logs traefik | grep -i crowdsec
docker logs crowdsec
docker exec crowdsec cscli decisions list
``` 
Comandos intersantes de crowdsec:

```bash
# Listado de decisiones tomadas
docker exec -it crowdsec cscli decisions list
# Métricas completas de crowdsec
docker exec -it crowdsec cscli metrics
# Unban IP
docker exec -it crowdsec cscli decisions delete -i x.x.x.x
```
### Protección de servicios con crowdsec  
Para proteger servicios tenemos dos opciones:  
1.- Configuración estática en dynamic_conf.yml  
2.- Etiquetas traefik en el servicio en cuestión.  
Por facilidad vamos a hacer la segunda opción, a través de etiquetas, así mantenemos más limpio el dynamic_conf.yml y traefik hace una carga en vivo sin reiniciar el proxy.  

Etiqueta:
```bash
# Ejemplo para Nextcloud. El resto de servicios sería exactamente igual
traefik.http.routers.nextcloud.middlewares=security-headers@file,crowdsec-bouncer@file
```

***   
Fuentes y enlaces de interés que ayudaran a complementar esta guía:  

[Que es S3 y por qué lo necesitas](https://atareao.es/podcast/que-es-s3-y-por-que-lo-necesitas-para-selfhosting/)    
[RustFS](https://rustfs.dev/)  
[Blog Rustfs](https://rustfs.dev/migrating-from-minio-to-rustfs-a-technical-guide/)  
[Minio S3](https://www.min.io/)  
[Duplicacy](https://duplicacy.com/)  
[Duplicati](https://duplicati.com/)  

