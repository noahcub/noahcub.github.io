---
title: Instalación de Outline Wiki
date: 2025-04-10 07:00:00 +0100
categories: [Nas]
tags: [software, config, unraid]     # TAG names should always be lowercase
img_path: /assets/pictures/Nas
author: <noah>
---
## Instalación de Outline

[Outline](https://www.getoutline.com/) es un gestor de conocimiento muy util y versatil.  

Para instalar Authentik debemos configurar 3 contenedores:  

![outline-1.png](outline-1.png)

La base de datos en postgresql, Redis y el propio servidor Outline. Es importante reseñar que Outline no permite la autenticación a través de usuario y contraseña, sino que pone a disposición del usuario varios servicios de autenticación. En este manual vamos a usar Authentik.  


**Redis**  
Este es muy sencillo. Usamos el contenedor de jj9987's Repository. Sencillísimo:

![outline-2.png](outline-2.png)


**Postgresql**    

![outline-3.png](outline-3.png)

Datos más relevantes:  

```bash
Name: postgresql17-outline   
Network Type: Custom  
Port: PostgreSQL access port: Nuestro puerto personalizado  
Path: /var/lib/postgresql/data: /mnt/user/appdata/outline/postgresql17  
Variable: POSTGRES_PASSWORD (Initial superuser password (required): XXXXXXXXXXXXXXXXXX  
Variable: POSTGRES_USER (Initial superuser name (default: postgres): USUARIO  
Variable: POSTGRES_DB (Initial database name (default: postgres): BASE_DATOS  
```

**Outline**   
  
Este contenedor lleva bastante trabajo hacerlo. Empezamos:  

![outline-4.png](outline-4.png)
![outline-5.png](outline-5.png)
![outline-6.png](outline-6.png)
![outline-7.png](outline-7.png)

Datos más relevantes:  

```bash
Name: outline
Repository: outlinewiki/outline:latest  
Network Type: Custom
Data Path: /mnt/user/appdata/outline/outline
###
### Importante la anotación del contenedor de Unraid: 
### Data Path For it to work chown 1001 /mnt/user/appdata/outline/
### SI NO EJECUTAMOS ESTE COMANDO PARA CAMBIAR LOS PERMISOS NO ARRANCARÁ NUESTRO CONTENEDOR
###
Port HTTP: puerto_acceso
SECRET_KEY (Container Variable: SECRET_KEY Generate a hex-encoded 32-byte random key. You could use `openssl rand -hex 32`): XXXXXXXXXXXXXXXXXXxx
UTILS_SECRET(Generate a hex-encoded 32-byte random key. You could use `openssl rand -hex 32`): XXXXXXXXXXXXXXXXXXXxxxxx
DATABASE_URL: postgres://USUARIO:PASSWOARD@192.168.IP_LOCAL_BASE_DATOS:PUERTO_BD/NOMBRE_BASE_DATOS
REDIS_URL: redis://IP_LOCAL_BASE_DATOS:PUERTO_REDIS
URL: https://outline.MIDOMINIO.com
FILE_STORAGE: local

# Datos generados en nuestro Authentik
OIDC_CLIENT_ID: Datos del OIDC Authentik
OIDC_CLIENT_SECRET: Datos del OIDC Authentik
OIDC_AUTH_URI: https://authentik.MIDOMINIO.com/application/o/authorize/
OIDC_TOKEN_URI: https://authentik.MIDOMINIO.com/application/o/token/
OIDC_USERINFO_URI: https://authentik.MIDOMINIO.com/application/o/wiki/end-session/
OIDC_USERNAME_CLAIM: preferred_username
OIDC_SCOPES: openid profile email

# Etiquetas traefik
traefik enable: true
traefik rule (Container Label: traefik.http.routers.outline.rule): Host(`outline.MIDOMINIO.com`)
traefik entrypoint (Container Label: traefik.http.routers.outline.entryPoints): https
Web UI (https)(Container Label: traefik.http.services.outline.loadbalancer.server.scheme): http
port (Container Label: traefik.http.services.outline.loadbalancer.server.port): 3000

```
Como se puede ver, no podemos configurar complemtamente nuestro contenedor hasta obtener los datos de OIDC Authentik.  
En la [web de Outline](https://docs.getoutline.com/s/hosting/doc/oidc-8CPBm6uC0I) tenemos información detallada para configurar correctamente el contenedor con Authentik. Tenemos un enlace a la web de Authentik que es donde realmente se obtienen los datos para acabar de configurar el contendor.
Vamos a ello. Según el [manual de integración de Authentik con Outline](https://docs.goauthentik.io/integrations/services/outline/) debemos crear una Aplicación y un proveedor de servicio:

Pulsamos en Crear Aplicación con proveedor con los datos de la imagen:

![outline-8.png](outline-8.png)

A la hora de continuar con el proveedor, seleccionamos **OAuth2/OpenID** y rellenamos los datos según la guía de Authentik para Outline:  

```bash
Note the Client ID,Client Secret, and slug values because they will be required later.
Set a Strict redirect URI to https://outline.MIDOMINIO.COM/auth/oidc.callback.
Select any available signing key.
```
![outline-9.png](outline-9.png)

El resto de datos se dejan por defecto.  
Por último podemos configurar políticas de usuario, etc.

Una vez finalizado, Authentik nos aporta los datos que necesitamos:

![outline-10.png](outline-10.png)

Con esto ya debería estar operativo.  

Si accedemos a nuestro outline **https://outline.MIDOMINIO.COM** vemos que debemos logearnos en Authentik para acceder al servicio:  

![outline-11.png](outline-11.png)

En mi caso, he creado un usuario estandar diferente del administrador para acceder a los servicios expuestos a través de Outline.  
  
**Es muy importante habilitar 2FA tanto en el usuario estandar como en el adminitrador**.

Como vemos, ya accedemos a nuestro panel de Outline:

![outline-12.png](outline-12.png)


***   
Fuentes y enlaces de interés que ayudaran a complementar esta guía:  
  
[https://www.getoutline.com/](https://www.getoutline.com/)  
[https://docs.goauthentik.io/integrations/services/outline/](https://docs.goauthentik.io/integrations/services/outline/)  
[https://danielpetrica.com/host-outline-with-docker-compose-and-traefik/](https://danielpetrica.com/host-outline-with-docker-compose-and-traefik/)  
[https://blog.gurucomputing.com.au/Doing%20More%20with%20Docker/Deploying%20Outline%20Wiki/](https://blog.gurucomputing.com.au/Doing%20More%20with%20Docker/Deploying%20Outline%20Wiki/)  

