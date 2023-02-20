---
title: Primeros pasos Fedora
date: 2023-02-02 18:00:00 +0100
categories: [System]
tags: [software, config, fedora]     # TAG names should always be lowercase
img_path: /assets/pictures/
author: <noah>
---
## Primeros pasos después de instalar Fedora

Configurar DNF para descargas más rápidas:
``` bash
sudo nano /etc/dnf/dnf.conf
```

y añadimos la siguiente línea:
``` bash
max_parallel_downloads=10
```

Realizamos una acutalización completa del sistema:
``` bash
sudo dnf upgrade -y
```

Habilitamos los respostorios de RPM Fusion:
``` bash
sudo dnf install https://download1.rpmfusion.org/free/fedora/rpmfusion-free-release-$(rpm -E %fedora).noarch.rpm
sudo dnf install https://download1.rpmfusion.org/nonfree/fedora/rpmfusion-nonfree-release-$(rpm -E %fedora).noarch.rpm
```

Después de habilitar RPM Fusion ya podemos instalar por ejemplo VLC
``` bash
sudo dnf install vlc
```

Añadimos el repositorio de Flathub:
``` bash
flatpak remote-add --if-not-exists flathub https://flathub.org/repo/flathub.flatpakrepo
```

En la instalación Fedora no pregunta por el nombre del host. Ahora podemos modificarlo:
``` bash
sudo hostnamectl set-hostname "New_Custom_Name"
```

Instalamos las aplicaciones esenciales para personalizar Gnome:
``` bash
sudo dnf install gnome-tweaks gnome-extensions-app
```
Instalamos también Extension Manager desde el administrador de Software de Gnome (es una aplicación Flatpak).

Habilitamos Maximizar y Minimizar en las ventanas:

![Foto.png](add-minimize-button-to-windows.png)

Habilitar/deshabilitar los informes de errores:
![Foto.png](automatic-problem-reporting-feature.png)

Configuraciones de energía:
![Foto.png](power-settings.png)

Habilitamos luz nocturna para reducir la fatiga visual:
![Foto.png](night-light-settings.png)

Ordenación de directorios antes de los ficheros:
![Foto.png](sort-folder-before-files.png)

Configuraciones de papelera de reciclaje:
![Foto.png](automatically-delete-trash-content.png)

***
Fuentes:  
[https://itsfoss.com/things-to-do-after-installing-fedora/](https://itsfoss.com/things-to-do-after-installing-fedora/)



