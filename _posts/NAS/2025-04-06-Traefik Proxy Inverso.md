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
  
Traefik necesita acceder al docker socket para monitorizar las etiquetas de los contenedores. Tenemos dos formas de hacerlo. IBRACORP lo explica muy bien. Vamos a instalar el **contendor dockersocket**. Se deja la configuración por defecto a excepción de la red, que seleccionamos **Custom: cloud**.  

Arrancamos el contenedor y para que traefik pueda acceder al docker socket por este medio tenemos que añadir una nueva variable:

![traefik-2](traefik-2.png)

También es importante realizar un cambio en el fichero traefik.yml. En el apartado **Docker provider**, debemos añadir el **endpoint: "tcp://dockersocket:2375"** según el recorte que tenemos a continuación:
  
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
  

``` bash
# Instructions: https://github.com/certbot/certbot/blob/master/certbot-dns-clou>
#
```

***   
Fuentes y enlaces de interés que ayudaran a complementar esta guía:  

[https://docs.ibracorp.io/traefik](https://docs.ibracorp.io/traefik)  
[https://diegoolivo.notion.site/Traefik-en-Unraid-45ed50c22bca4708adad442e1e92f2c4#c83b3526c9c5456ab747d3da6db57da4](https://diegoolivo.notion.site/Traefik-en-Unraid-45ed50c22bca4708adad442e1e92f2c4#c83b3526c9c5456ab747d3da6db57da4)  
[https://github.com/UnRAIDES/Scripts/tree/main/olivoa/traefik](https://github.com/UnRAIDES/Scripts/tree/main/olivoa/traefik)  
[https://www.reddit.com/r/Authentik/comments/1dio0cw/authentik_traefik_paperlessngx_oauth/?tl=es-es](https://www.reddit.com/r/Authentik/comments/1dio0cw/authentik_traefik_paperlessngx_oauth/?tl=es-es)  
[https://github.com/brokenscripts/authentik_traefik?tab=readme-ov-file](https://github.com/brokenscripts/authentik_traefik?tab=readme-ov-file)  
