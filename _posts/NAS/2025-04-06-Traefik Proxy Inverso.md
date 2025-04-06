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
Traefik Dashboard Subdomain (Container Label: traefik.http.routers.api.rule): Host(`traefik.MIDOMINIO.com`)  
Traefik API (Container Label: traefik.http.routers.api.service): api@internal  
Traefik Entrypoint (Container Label: traefik.http.routers.api.entryPoints): https  
Enable Traefik (Dashboard) (Container Label: traefik.enable): true  


Con esto el contenedor ya está configurado. Según le damos a aceptar **no arrancará** porque tenemos que crear los ficheros de configuración.  


``` bash
# Instructions: https://github.com/certbot/certbot/blob/master/certbot-dns-clou>
# Replace with your values

# With global api key:
dns_cloudflare_email = micorreo@correo.com
dns_cloudflare_api_key = xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx

# With token (comment out both lines above and uncomment below):
# dns_cloudflare_api_token = xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
```
Como se observa en el fichero se puede realizar el registro por api key o por token. En mi caso está hecho con api key.  
Reiniciamos el contenedor y debería funcionar correctamente el registro de los certificados.  

![swag-cloudflare-5](swag-cloudflare-5.png)
![swag-cloudflare-6](swag-cloudflare-6.png)
![swag-cloudflare-7](swag-cloudflare-7.png)
![swag-cloudflare-8](swag-cloudflare-8.png)

Si probamos la conexión debería funcionar:

![swag-cloudflare-9.png](swag-cloudflare-9.png)

## Configuración en el panel de control de Cloudflare para los subdominios  

En el apartado DNS -> Registros debe quedar de la siguiente forma:  

![swag-cloudflare-10.png](swag-cloudflare-10.png)

El primer registro se genera solo con el contenedor Cloudflare-DDNS y los otros los añado yo.  
El subdominio nextcloud es para el acceso a nuestro Nextcloud a través de internet.  

***   
Fuentes y enlaces de interés que ayudaran a complementar esta guía:  

[https://docs.ibracorp.io/traefik](https://docs.ibracorp.io/traefik)  
[https://diegoolivo.notion.site/Traefik-en-Unraid-45ed50c22bca4708adad442e1e92f2c4#c83b3526c9c5456ab747d3da6db57da4](https://diegoolivo.notion.site/Traefik-en-Unraid-45ed50c22bca4708adad442e1e92f2c4#c83b3526c9c5456ab747d3da6db57da4)  
[https://github.com/UnRAIDES/Scripts/tree/main/olivoa/traefik](https://github.com/UnRAIDES/Scripts/tree/main/olivoa/traefik)  
[https://www.reddit.com/r/Authentik/comments/1dio0cw/authentik_traefik_paperlessngx_oauth/?tl=es-es](https://www.reddit.com/r/Authentik/comments/1dio0cw/authentik_traefik_paperlessngx_oauth/?tl=es-es)  
[https://github.com/brokenscripts/authentik_traefik?tab=readme-ov-file](https://github.com/brokenscripts/authentik_traefik?tab=readme-ov-file)  
