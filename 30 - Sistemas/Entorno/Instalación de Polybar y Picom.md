---
aliases:
  - Polybar Picom Instalacion
  - BSPWM Compositor Bar
tags:
  - linux/customizacion
  - linux/administracion
estado: 🟢 Terminado
---
# Instalación de Polybar y Picom

Ya con listo, vamos a realizar la instalación de la siguiente forma:

## Instalación de Polybar

Para instalar Polybar, vamos a hacer lo siguiente:

```bash
sudo apt install polybar
```

> [!info] Polybar: Barra de Estado Modular
> Polybar es una barra de estado altamente personalizable y modular diseñada para entornos de escritorio Linux, especialmente para gestores de ventanas tipo *tiling* como BSPWM. Permite mostrar información variada como el uso de CPU/RAM, estado de la red, volumen, fecha/hora, y controlar reproductores de música, entre otros. Su configuración se realiza mediante archivos de texto plano, ofreciendo una gran flexibilidad para adaptar su apariencia y funcionalidad a las necesidades del usuario.

## Instalación de Picom

Para instalar Picom, vamos a realizar los siguientes comandos dentro de la carpeta `descargas`:

```bash
sudo apt install libconfig-dev libdbus-1-dev libegl-dev libev-dev libgl-dev libepoxy-dev libpcre2-dev libpixman-1-dev libx11-xcb-dev libxcb1-dev libxcb-composite0-dev libxcb-damage0-dev libxcb-dpms0-dev libxcb-glx0-dev libxcb-image0-dev libxcb-present-dev libxcb-randr0-dev libxcb-render0-dev libxcb-render-util0-dev libxcb-shape0-dev libxcb-util-dev libxcb-xfixes0-dev libxext-dev meson ninja-build uthash-dev

git clone https://github.com/yshui/picom
cd picom

meson setup --buildtype=release build
ninja -C build
ninja -C build install
```

> [!info] Picom: Compositor Ligero para Xorg
> Picom es un compositor ligero para el sistema de ventanas Xorg, conocido por ser un *fork* de Compton. Su función principal es eliminar el *tearing* (desgarro de pantalla), añadir efectos visuales como transparencia, sombras, *fading* y animaciones de ventanas. Es esencial para una experiencia visual fluida y moderna en gestores de ventanas minimalistas que no incluyen su propio compositor. Las dependencias `lib*-dev` son librerías de desarrollo necesarias para compilar Picom desde el código fuente, ya que interactúa directamente con el sistema Xorg y sus extensiones (como XCB para comunicación entre cliente y servidor X, GL para renderizado, etc.).

> [!tip] Meson y Ninja: Sistemas de Construcción Modernos
> **Meson** es un sistema de construcción de proyectos de código abierto diseñado para ser rápido y fácil de usar. Se enfoca en la velocidad y la portabilidad, generando archivos de construcción para *backends* como Ninja. **Ninja** es un sistema de construcción de bajo nivel que se centra en la velocidad. A diferencia de Make, Ninja no está diseñado para ser escrito a mano, sino para ser generado por sistemas de construcción de alto nivel como Meson. Juntos, ofrecen un proceso de compilación eficiente y optimizado.

Una vez completado esto, vamos a escribir el siguiente comando, el cual cerrará la sesión: