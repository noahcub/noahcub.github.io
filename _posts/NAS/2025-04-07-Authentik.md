---
title: Instalación de Authentik e integración con Traefik
date: 2025-04-07 07:00:00 +0100
categories: [Nas]
tags: [software, config, unraid]     # TAG names should always be lowercase
img_path: /assets/pictures/Nas
author: <noah>
---
## Instalación de Authentik

[Authentik](https://goauthentik.io/) es un proveedor de identidad de código abierto y autohospedado.  

Para instalar Authentik debemos configurar 4 contenedores:  

![authentik-1.png](authentik-1.png)

La base de datos en postgresql, Redis, authentik-server y authentik-worker.  
Vamos a seguir los pasos del docker-compose pero modificando para hacerlo a través de las plantillas de Unraid:

```bash
services:
  authentik-postgresql:
    container_name: authentik-postgresql
    image: docker.io/library/postgres:16-alpine
    restart: unless-stopped
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -d $${POSTGRES_DB} -U $${POSTGRES_USER}"]
      start_period: 20s
      interval: 30s
      retries: 5
      timeout: 5s
    volumes:
      - database:/var/lib/postgresql/data
    environment:
      POSTGRES_PASSWORD: ${PG_PASS:?database password required}
      POSTGRES_USER: ${PG_USER:-authentik}
      POSTGRES_DB: ${PG_DB:-authentik}
    env_file:
      - .env

  authentik-redis:
    container_name: authentik-redis
    image: docker.io/library/redis:alpine
   #command: --save 60 1 --loglevel warning
    restart: unless-stopped
    healthcheck:
      test: ["CMD-SHELL", "redis-cli ping | grep PONG"]
      start_period: 20s
      interval: 30s
      retries: 5
      timeout: 3s
    #volumes:
    #  - redis:/data

  authentik-server:
    container_name: authentik-server
    image: ${AUTHENTIK_IMAGE:-ghcr.io/goauthentik/server}:${AUTHENTIK_TAG:-2025.2.3}
    restart: unless-stopped
    command: server
    environment:
      AUTHENTIK_REDIS__HOST: authentik-redis
      AUTHENTIK_POSTGRESQL__HOST: authentik-postgresql
      AUTHENTIK_POSTGRESQL__USER: ${PG_USER:-authentik}
      AUTHENTIK_POSTGRESQL__NAME: ${PG_DB:-authentik}
      AUTHENTIK_POSTGRESQL__PASSWORD: ${PG_PASS}
      AUTHENTIK_REDIS__DB: 1
    volumes:
      - /mnt/user/appdata/authentik/media:/media
      - /mnt/user/appdata/authentik/custom-templates:/templates
    env_file:
      - .env
    ports:
      - "${COMPOSE_PORT_HTTP:-9000}:9000"
      - "${COMPOSE_PORT_HTTPS:-9443}:9443"

    labels:
      traefik.enable: true
      traefik.http.routers.authentik.entryPoints: https
      traefik.http.routers.authentik.rule: Host(`authentik.MIDOMINIO.com`) || HostRegexp(`{subdomain:[A-Za-z0-9](?:[A-Za-z0-9\-]{0,61}[A-Za-z0-9])?}.MIDOMINIO.com`) && PathPrefix(`/outpost.goauthentik.io/`)

    depends_on:
      authentik-postgresql:
        condition: service_healthy
      authentik-redis:
        condition: service_healthy

  authentik-worker:
    container_name: authentik-worker
    image: ${AUTHENTIK_IMAGE:-ghcr.io/goauthentik/server}:${AUTHENTIK_TAG:-2025.2.3}
    restart: unless-stopped
    command: worker
    environment:
      AUTHENTIK_REDIS__HOST: authentik-redis
      AUTHENTIK_POSTGRESQL__HOST: authentik-postgresql
      AUTHENTIK_POSTGRESQL__USER: ${PG_USER:-authentik}
      AUTHENTIK_POSTGRESQL__NAME: ${PG_DB:-authentik}
      AUTHENTIK_POSTGRESQL__PASSWORD: ${PG_PASS}
      AUTHENTIK_REDIS__DB: 1
    user: root
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - /mnt/user/appdata/authentik/media:/media
      - /mnt/user/appdata/authentik/certs:/certs
      - /mnt/user/appdata/authentik/custom-templates:/templates
    env_file:
      - .env
    depends_on:
      authentik-postgresql:
        condition: service_healthy
      authentik-redis:
        condition: service_healthy

volumes:
  database:
    driver: local
  redis:
    driver: local
networks:
  default:
    name: MI_RED_DOCKER
    external: true
```

**Postgresql**    
Name: authentik-postgresql16  
Network Type: Custom  
Port: PostgreSQL access port: Nuestro puerto personalizado  
Path: /var/lib/postgresql/data:/mnt/user/appdata/authentik/postgresql16  
Variable: POSTGRES_PASSWORD (Initial superuser password (required): XXXXXXXXXXXXXXXXXX  
Variable: POSTGRES_USER (Initial superuser name (default: postgres): USUARIO  
Variable: POSTGRES_DB (Initial database name (default: postgres): BASE_DATOS  

![authentik-2.png](authentik-2.png)
![authentik-3.png](authentik-3.png)

**Redis**  
Este es muy sencillo. Usamos el contenedor de A75G's Repository. Es importante leer los requerimientos adicionales:

```bash
Additional Requirements
chown -R 1001:1001 /mnt/user/appdata/redis/
```

![authentik-4.png](authentik-4.png)
![authentik-5.png](authentik-5.png)


**Authentik-server**  
Tanto el worker como el server, se instalará a través de las plantillas de IBRACORP.
![authentik-6.png](authentik-6.png)
![authentik-7.png](authentik-7.png)
![authentik-8.png](authentik-8.png)
![authentik-9.png](authentik-9.png)
![authentik-10.png](authentik-10.png)

Valores importantes en la configuración:  
Name: authentik-server  
Repository:beryju/authentik:latest  
Network Type: custom  
AUTHENTIK_PORT_HTTP: xxxxx  
AUTHENTIK_PORT_HTTPS: xxxxxx  
Redis Host:authentik-redis  
PostgreSQL Host:authentik-postgresql16    
PostgreSQL DB User:usuario-base-datos  
PostgreSQL DB Name:base-datos  
PostgreSQL DB Password:pass-basedatos  
APP Key (Container Variable: AUTHENTIK_SECRET_KEY https://passwordsgenerator.net/): xxxxxxxxxxxxxxxxx  
Redis Password:xxxxxxxxxxxxxxxxxxx  
Docker Socket:/var/run/docker.sock  
traefik enable:true  
traefik entrypoints:https  
traefik rule:Host(`authentik.MIDOMINIO.com`) || HostRegexp(`{subdomain:[A-Za-z0-9](?:[A-Za-z0-9\-]{0,61}[A-Za-z0-9])?}.MIDOMINIO.com`) && PathPrefix(`/outpost.goauthentik.io/`)   

**Authentik-worker**  
En el caso del worker no hace falta las capturas porque es muy sencilla. Estos son los datos básicos:  

Name: authentik-worker  
Repository:beryju/authentik:latest  
Network Type: custom    
Redis Host:authentik-redis  
PostgreSQL Host:authentik-postgresql16    
PostgreSQL DB User:usuario-base-datos  
PostgreSQL DB Name:base-datos  
PostgreSQL DB Password:pass-basedatos  
Secret Key (Container Variable: AUTHENTIK_SECRET_KEY - SAME AS THE AUTHENTIK SERVER): XXXXXXXXXXXXXXXX   
Backups: /mnt/user/appdata/authentik/worker/  
Media: /mnt/user/appdata/authentik/worker/  
Certs: /mnt/user/appdata/authentik/worker/  
Docker Socket:/var/run/docker.sock  
Redis Password: XXXXXXXXXXXXXXXXXXXX  
  

Con esto ya debería estar operativo.  

Para la configuración inicial, accedemos al siguiente enlace **https://<server_IP>/if/flow/initial-setup/** y seguimos las instrucciones. Es muy importante activar el 2FA en todas las cuentas que usemos en Authentik.  
  
Si accedemos al panel de control de traefik veremos que yá está operativo nuestro Authentik.

![authentik-11.png](authentik-11.png)

***   
Fuentes y enlaces de interés que ayudaran a complementar esta guía:  

[https://docs.ibracorp.io/authentik](https://docs.ibracorp.io/authentik)  
[https://github.com/brokenscripts/authentik_traefik?tab=readme-ov-file](https://github.com/brokenscripts/authentik_traefik?tab=readme-ov-file)  
[https://goauthentik.io/](https://goauthentik.io/)  
