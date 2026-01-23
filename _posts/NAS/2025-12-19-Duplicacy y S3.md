---
title: Backups con Duplicacy y S3
description: Backups con Duplicacy y S3
date: 2025-12-19 13:40:36 +01:00
categories: [Nas]
tags: [software, config, unraid]     # TAG names should always be lowercase
img_path: /assets/pictures/Nas
author: <noah>
---

## Configuración de Backups con Duplicacy  
  
En mi servidor Unraid tengo instalado un contenedor de Duplicacy que se encarga de hacer Backups de los datos importantes en mi cuenta de google drive y en un servidor deslocalizado que tiene instalada una instancia de S3 para los backups.

### Instalación de Duplicacy  
  
Me he decidido por Duplicacy porque tiene un entorno gráfico muy sencillo y cada día busco más comodidad a la hora de hacer las cosas.  
Busqué muchas opiniones de **Duplicacy vs Duplicati**. Llevo varios años usando Duplicacy pero debo decir que la nueva imagen que le han dado a Duplicati está genial.  
Duplicacy es de pago. El primer año se pagan 20$ y los sucesivos 10$. Me parece un precio muy económico por la labor que hace, aunque hay que tener en cuenta que Duplicati no tiene coste alguno.  
La instalación en UNRAID es muy sencilla:
![duplicacy-1.png](duplicacy-1.png)

El resto de opciones yo las dejo por defecto:
![duplicacy-2.png](duplicacy-2.png)

Accedemos a nuestro entorno web: http://192.168.10.55:38975/.  
En el menu de la izquierda tenemos los distintos elementos.   


### Creación de un Storage para Backup en GoogleDrive
Seguimos las instrucciones que nos dice:  
Se requiere un archivo de token de acceso emitido por Google para que Duplicacy pueda realizar copias de seguridad en Google Drive. Visite esta página para autorizar a Duplicacy a acceder a los archivos de su cuenta de Google Drive. Se descargará el archivo de token de acceso y deberá proporcionar la ruta en el campo "Archivo de token" que aparece a continuación.  

El campo "Unidad compartida" puede estar vacío si accede a "Mi unidad" en lugar de a una unidad compartida.

![duplicacy-3.png](duplicacy-3.png)

### Creación de un Storage para Backup en S3
Esta parte es un poco más compleja pero funciona de la siguiente manera:
![duplicacy-4.png](duplicacy-4.png)
![duplicacy-5.png](duplicacy-5.png)

Los datos los obtenemos de nuestro servidor S3. Lo vemos en el siguiente punto.

### Configurar nuestro contenedor S3 en el servidor remoto

Tenemos dos opciones: usar MINIO o RUSTFS. He estado usando MINIO durante mucho tiempo hasta que un buen día los desarrolladores se cargaron la interfaz web. Recientemente el proyecto ha entrado en modo mantenimiento y era cuestión de tiempo que tuviera que hacer una migración. He instalado RUSTFS y tendré los dos sistemas funcionando en paralelo. Si va todo como espero, me cambiaré definitivamente a RUSTFS.  
RUSTFS lo descubrí gracias a que soy un fan del podcast [Atareao con Linux](https://atareao.es/).

**docker-compose de minio**
``` bash
services:
  minio:
    image: minio/minio:RELEASE.2025-04-22T22-12-26Z
    container_name: minio
    ports:
      - "9010:9000"
      - "9011:9001"
    environment:
      MINIO_ROOT_USER: minioadmin
      MINIO_ROOT_PASSWORD: XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX
    restart: unless-stopped

    volumes:
      - /home/Backups/minio:/data
    command: server /data --console-address ":9001"
```

**docker-compose de rustfs**
``` bash
services:
  rustfs:
    image: rustfs/rustfs
    container_name: rustfs
    restart: unless-stopped
    init: true
    ports:
      - "9020:9000" # S3 API port
      - "9021:9001" # Console port
    volumes:
      - /home/Backups/rustfs:/data
    environment:
      RUSTFS_ACCESS_KEY: rustfsadmin
      RUSTFS_SECRET_KEY: xxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
      RUSTFS_ADDRESS: 0.0.0.0:9000
      RUSTFS_CONSOLE_ADDRESS: 0.0.0.0:9001
      RUSTFS_EXTERNAL_ADDRESS: 9000
      RUSTFS_CONSOLE_ENABLE: true
    healthcheck:
      test:
        - CMD
        - sh
        - -c
        - curl -f http://localhost:9000/health && curl -f
          http://localhost:9001/rustfs/console/health
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 40s
```

Arrancamos el contenedor de RUSTFS y accedemos con las credenciales que hemos configurado.  
Lo primero que vamos a hacer es crear un Access Key:

![duplicacy-6.png](duplicacy-6.png)  
![duplicacy-7.png](duplicacy-7.png)

Estos datos de Access Key son los que vamos a añadir en la configuración del Backup de Duplicacy del punto anterior.  
Después creamos un bucket. Esto es muy sencillo, simplemente le damos un nombre y listo.
![duplicacy-8.png](duplicacy-8.png)

Y añadimos el nombre del bucket a la configuración de Duplicacy.

### Migración de MINIO a RUSTFS

En este caso la cosa se complica un poco más porque queremos mantener las copias de seguridad que hemos hecho desde MINIO con nuestro nuevo sistema de RUSTFS.  
EN el [blog de Rustfs](https://rustfs.dev/migrating-from-minio-to-rustfs-a-technical-guide/) nos dan una solución muy sencilla: 

``` bash
# Con 4 comandos lo tenemos hecho
# Accedemos a una terminal del contenedor MINIO
docker exec -it minio /bin/bash
# Creamos el primer alias de nuestro minio actualmente operativo
mc alias set minio-old http://100.105.100.11:9010 USER_KEY_MINIO PASS_KEY_MINIO
# Creamos un segundo alias de nuestro rustfs reción configurado
# Es importante tener en cuenta que hay que crear un bucket en RUSTFS con el mismo nombre que en MINIO
mc alias set rustfs-new http://100.105.100.11:9020 USER_KEY_RUSTFS PASS_KEY_RUSTFS
# Probamos que tenemos conectividad y permisos con el nuevo servidor
mc ls rustfs-new
# Hacemos una simulación de transferencia de los datos
mc mirror --dry-run minio-old/sanmi rustfs-new/sanmi
# Ejecutamos la transferencia de forma real
# Es importante hacer esto con ningun backup en funcionamiento, sino habría que usar otro comando que explica en el blog.
mc mirror minio-old/sanmi rustfs-new/sanmi
# Verificamos que el mirror sea exactamente igual:
mc diff minio-old/sanmi rustfs-new/sanmi
# Si este comando no devuelve nada es que todo está correcto.
```

Una vez hecha la migración creamos un nuevo endpoint de S3 en Duplicacy o modificamos el que ya teníamos.

### Backups en Duplicacy

![duplicacy-9.png](duplicacy-9.png)

### Schedule en Duplicacy

Muy sencillo también. Programamos cada tarea. 
![duplicacy-10.png](duplicacy-10.png)

Yo tengo distintas tareas programadas: Backups diarios, Backups cada 8 horas para Outline y Backups semanales para Nextclod, Prune (limpieza) y Check de los backups.

### Restore en Duplicacy

Esta opción también es muy intuitiva. No tiene mucho que explicar.

***   
Fuentes y enlaces de interés que ayudaran a complementar esta guía:  

[Que es S3 y por qué lo necesitas](https://atareao.es/podcast/que-es-s3-y-por-que-lo-necesitas-para-selfhosting/)    
[RustFS](https://rustfs.dev/)  
[Blog Rustfs](https://rustfs.dev/migrating-from-minio-to-rustfs-a-technical-guide/)  
[Minio S3](https://www.min.io/)  
[Duplicacy](https://duplicacy.com/)  
[Duplicati](https://duplicati.com/)  

