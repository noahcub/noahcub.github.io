---
title: Deshabilitar renderizado GPU en Visual Studio Code y Microsoft Teams
date: 2023-02-05 08:00:00 +0100
categories: [Software]
tags: [software, programming]     # TAG names should always be lowercase
img_path: /pictures/
author: <noah>
---
## Como deshabilitar renderizdo de GPU en algunos programas

Algunos programas generan errores de visualización debido al renderizado de GPU con drivers Nvidia. En la mayoría lo podemos deshabilitar:

**Visual Studio Code**

***Primera opcion***

Buscamos el lanzador de Visual Studio en Gnome:

Normalmente en 
```bash
/usr/share/applications/code.desktop
```
Editamos el fichero cambiando:
```bash
Exec=/usr/share/code/code --unity-launch %F
```
por
```bash
Exec=/usr/share/code/code --disable-gpu --unity-launch %F
```

***Segunda opcion***

- Abrimos Command Palette en Visual Studio (Ctrl+Shift+P).

- `Preferences: Configure Runtime Arguments` Este comando abre el fichero `argv.json` para configurar argumentos en tiempo de ejecución.

- Añadimos lo siguiente:
```bash
"disable-hardware-acceleration": true 
```
- Reiniciamos Visual Studio    

***Tercera opción***

Editamos directamente el fichero 
```bash
~/.vscode/argv.json
```
Añadimos lo mismo que en la opcion dos:
```bash
"disable-hardware-acceleration": true 
```

**Microsoft Teams**

Editamos directamente el fichero  de configuracion desktop-config.json

```bash
~/.config/Microsoft/Microsoft Teams/desktop-config.json
```
Cambiamos la opción "disableGpu":false por true:
```bash
"disableGpu":true
```

***
Fuentes:  

