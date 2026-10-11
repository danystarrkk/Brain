---
aliases:
  - Blue Sky Polybar
  - Configuración Polybar
tags:
  - linux/customizacion
estado: 🟢 Terminado
---
# Instalación y Configuración de Blue Sky Polybar

Una vez configurado, procederemos a ejecutar el siguiente comando en el directorio de descargas para instalar y configurar el tema `blue-sky` para Polybar.

> [!info] **Polybar**
> Polybar es una barra de estado ligera y altamente personalizable diseñada para sistemas Linux. Se utiliza comúnmente con gestores de ventanas de mosaico (tiling window managers) como `bspwm`, `i3`, `AwesomeWM`, etc., para mostrar información del sistema (CPU, RAM, red), notificaciones, controles de volumen, y más, proporcionando una interfaz de usuario limpia y funcional.

```bash
git clone https://github.com/VaughnValle/blue-sky.git
cd blue-sky/polybar/ 
cp -r * ~/.config/polybar
echo '~/.config/polybar/./launch.sh &' >> ~/.config/bspwm/bspwmrc
sudo cp fonts/* /usr/share/fonts/truetype
fc-cache -v
```

## Explicación Detallada de los Comandos

1.  `git clone https://github.com/VaughnValle/blue-sky.git`
    > [!info] **`git clone`**
    > Este comando descarga un repositorio de Git desde una URL remota a tu máquina local. En este caso, clona el repositorio `blue-sky` que contiene archivos de configuración y temas. Crea una copia local completa del repositorio, incluyendo todo su historial de versiones.

2.  `cd blue-sky/polybar/`
    > [!info] **`cd`**
    > El comando `cd` (change directory) se utiliza para navegar entre directorios en el sistema de archivos. Aquí, cambia el directorio de trabajo actual al subdirectorio `polybar` dentro del repositorio `blue-sky` recién clonado, donde se encuentran los archivos de configuración específicos de Polybar.

3.  `cp -r * ~/.config/polybar`
    > [!info] **`cp -r`**
    > El comando `cp` se usa para copiar archivos y directorios. La opción `-r` (recursivo) es crucial aquí, ya que permite copiar directorios y su contenido. Este comando copia todos los archivos y subdirectorios (`*`) del directorio actual (`blue-sky/polybar/`) al directorio `~/.config/polybar`. El directorio `~/.config/` es una ubicación estándar para almacenar archivos de configuración de usuario en sistemas Linux.

4.  `echo '~/.config/polybar/./launch.sh &' >> ~/.config/bspwm/bspwmrc`
    > [!warning] **Persistencia y `bspwmrc`**
    > Este comando añade una línea al final del archivo `~/.config/bspwm/bspwmrc`. El archivo `bspwmrc` es el script de configuración principal para el gestor de ventanas `bspwm`. Se ejecuta cada vez que `bspwm` se inicia.
    > *   `echo '~/.config/polybar/./launch.sh &'` imprime la cadena de texto.
    > *   `>>` es un operador de redirección que añade la salida del comando `echo` al final del archivo especificado (`~/.config/bspwm/bspwmrc`).
    > *   `&` al final del comando `launch.sh` significa que el script se ejecutará en segundo plano, permitiendo que `bspwm` continúe su inicio sin esperar a que Polybar termine. El script `launch.sh` es responsable de iniciar las instancias de Polybar para el tema `blue-sky`.

5.  `sudo cp fonts/* /usr/share/fonts/truetype`
    > [!tip] **Instalación de Fuentes del Sistema**
    > Este comando copia todos los archivos de fuentes (`fonts/*`) del directorio actual (que se asume que contiene las fuentes necesarias para el tema `blue-sky`) al directorio `/usr/share/fonts/truetype`. Este es un directorio estándar del sistema donde se almacenan las fuentes TrueType para que estén disponibles para todas las aplicaciones y usuarios del sistema. `sudo` es necesario porque `/usr/share/fonts/` es un directorio del sistema que requiere permisos de superusuario para escribir.

6.  `fc-cache -v`
    > [!info] **Actualización de la Caché de Fuentes**
    > El comando `fc-cache` se utiliza para construir o reconstruir las cachés de fuentes del sistema. Después de añadir nuevas fuentes a un directorio del sistema, es necesario ejecutar `fc-cache` para que el sistema las detecte y las haga disponibles para las aplicaciones. La opción `-v` (verbose) muestra información detallada sobre el proceso de escaneo y construcción de la caché.