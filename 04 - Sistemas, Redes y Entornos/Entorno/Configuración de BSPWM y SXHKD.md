---
aliases: ["bspwm", "sxhkd", "configuracion window manager"]
tags: ["linux/customizacion", "scripting/bash", "linux/administracion"]
estado: 🟢 Terminado
---
# Configuración de BSPWM y SXHKD

Primeramente, ejecutaremos el siguiente comando para instalar las dependencias necesarias:

```Bash
sudo apt install build-essential git vim xcb libxcb-util0-dev libxcb-ewmh-dev libxcb-randr0-dev libxcb-icccm4-dev libxcb-keysyms1-dev libxcb-xinerama0-dev libasound2-dev libxcb-xtest0-dev libxcb-shape0-dev
```

> [!info] Dependencias de compilación y desarrollo
> Este comando instala `build-essential` (colección de paquetes para compilar software, incluyendo `gcc`, `g++` y `make`), `git` (control de versiones), `vim` (editor de texto), y una serie de librerías `libxcb-*dev` que son fundamentales para el desarrollo y funcionamiento de gestores de ventanas basados en XCB (X-C Binding), como `bspwm`. Estas librerías proporcionan las interfaces de programación para interactuar con el servidor Xorg de manera eficiente. `libasound2-dev` es para soporte de audio.

Posteriormente, aplicaremos una actualización del sistema con el comando `apt update`. Acto seguido, diríjanse a la carpeta de descargas de su equipo y descarguen los proyectos `bspwm` y `sxhkd` con los siguientes comandos:

```Bash
git clone https://github.com/baskerville/bspwm.git
git clone https://github.com/baskerville/sxhkd.git
```

> [!tip] Clonación de repositorios Git
> `git clone` descarga una copia completa de un repositorio Git remoto al directorio local. `bspwm` (Binary Space Partitioning Window Manager) es un gestor de ventanas de mosaico que representa las ventanas como las hojas de un árbol binario completo. `sxhkd` (Simple X Hotkey Daemon) es un demonio de atajos de teclado que reacciona a eventos de teclado y ejecuta comandos. Ambos son componentes clave para una experiencia de escritorio minimalista y altamente personalizable.

Para instalar cada uno de estos, deben ingresar a ambos directorios por separado y ejecutar los comandos `make` y `sudo make install`.

> [!info] Proceso de compilación e instalación
> El comando `make` compila el código fuente del proyecto utilizando el `Makefile` presente en el directorio. `sudo make install` copia los binarios compilados y otros archivos necesarios a las ubicaciones estándar del sistema (ej. `/usr/local/bin`, `/usr/local/share`), requiriendo privilegios de superusuario.

## Configuración inicial de BSPWM y SXHKD

Luego de esto, comenzaremos a configurar los archivos propios tanto de `sxhkd` como de `bspwm`. Para ello, ejecutaremos los siguientes comandos:

```bash
mkdir ~/.config/{bspwm,sxhkd}
```

> [!tip] Estructura de directorios de configuración
> El comando `mkdir -p ~/.config/{bspwm,sxhkd}` crea los directorios `bspwm` y `sxhkd` dentro de `~/.config/`. Esta es la ubicación estándar (XDG Base Directory Specification) para archivos de configuración de usuario, lo que permite mantener el directorio `home` más limpio.

Posteriormente, ejecutamos el siguiente comando:

```bash
cp ~/Descargas/bspwm/examples/bspwmrc ~/.config/bspwm/ && cp ~/Descargas/bspwm/examples/sxhkdrc ~/.config/sxhkd/
```

> [!info] Copia de archivos de configuración de ejemplo
> Este comando copia los archivos de configuración de ejemplo (`bspwmrc` y `sxhkdrc`) desde los directorios de los repositorios clonados a las ubicaciones de configuración del usuario. `bspwmrc` es el script principal de configuración de `bspwm` que se ejecuta al iniciar el gestor de ventanas, y `sxhkdrc` define los atajos de teclado para `sxhkd`. El `&&` asegura que el segundo comando solo se ejecute si el primero fue exitoso.

## Instalación de paquetes adicionales

A continuación, instalaremos algunos paquetes adicionales:

```bash
sudo apt install kitty rofi imagemagick feh wmname
```

> [!info] Herramientas complementarias
> *   `kitty`: Un emulador de terminal rápido y rico en características, acelerado por GPU.
> *   `rofi`: Un lanzador de aplicaciones, selector de ventanas y reemplazo de `dmenu` altamente configurable.
> *   `imagemagick`: Un conjunto de herramientas para crear, editar, componer o convertir imágenes bitmap. Útil para capturas de pantalla o manipulación de fondos.
> *   `feh`: Un visor de imágenes ligero que también puede usarse para establecer el fondo de escritorio (wallpaper).
> *   `wmname`: Una pequeña utilidad para establecer el nombre del gestor de ventanas, útil para algunas aplicaciones Java que requieren un gestor de ventanas "reparenting".

## Configuración del archivo sxhkdrc

La configuración final del archivo `sxhkdrc` es la siguiente:

```bash
#
# wm independent hotkeys
#

# terminal emulator
super + Return
	/usr/bin/kitty

# program launcher
super + d
	/usr/bin/rofi -show run

# make sxhkd reload its configuration files:
super + Escape
	pkill -USR1 -x sxhkd

#
# bspwm hotkeys
#

# quit/restart bspwm
super + shift + {q,r}
	bspc {quit,wm -r}

# close and kill
super + {_,shift + }q
	bspc node -{c,k}

# alternate between the tiled and monocle layout
super + m
	bspc desktop -l next

# send the newest marked node to the newest preselected node
super + y
	bspc node newest.marked.local -n newest.!automatic.local

# swap the current node and the biggest window
super + g
	bspc node -s biggest.window

#
# state/flags
#

# set the window state
super + {t,shift + t,s,f}
	bspc node -t {tiled,pseudo_tiled,floating,fullscreen}

# set the node flags
super + ctrl + {m,x,y,z}
	bspc node -g {marked,locked,sticky,private}

#
# focus/swap
#

# focus the node in the given direction
super + {_,shift + }{Left,Down,Up,Right}
	bspc node -{f,s} {west,south,north,east}

# focus the node for the given path jump
super + {p,b,comma,period}
	bspc node -f @{parent,brother,first,second}

# focus the next/previous window in the current desktop
super + {_,shift + }c
	bspc node -f {next,prev}.local.!hidden.window

# focus the next/previous desktop in the current monitor
super + bracket{left,right}
	bspc desktop -f {prev,next}.local

# focus the last node/desktop
super + {grave,Tab}
	bspc {node,desktop} -f last

# focus the older or newer node in the focus history
super + {o,i}
	bspc wm -h off; \
	bspc node {older,newer} -f; \
	bspc wm -h on

# focus or send to the given desktop
super + {_,shift + }{1-9,0}
	bspc {desktop -f,node -d} '^{1-9,10}'

#
# preselect
#

# preselect the direction
super + ctrl + alt + {Left,Down,Up,Right}
	bspc node -p {west,south,north,east}

# preselect the ratio
super + ctrl + {1-9}
	bspc node -o 0.{1-9}

# cancel the preselection for the focused node
super + ctrl + alt + space
	bspc node -p cancel

# cancel the preselection for the focused desktop
super + ctrl + shift + space
	bspc query -N -d | xargs -I id -n 1 bspc node id -p cancel

#
# move/resize
#

# move a floating window
super + shift + {Left,Down,Up,Right}
	bspc node -v {-20 0,0 20,0 -20,20 0}

# Custom resize

super + alt + {Left,Down,Up,Right}
	/home/stark/.config/bspwm/scripts/bspwm_resize {west,south,north,east}
```

> [!info] Sintaxis de sxhkdrc y comandos bspc
> El archivo `sxhkdrc` define los atajos de teclado. La sintaxis es `[modificador +] tecla` seguido de un comando.
> *   `super`: Generalmente se refiere a la tecla "Windows" o "Meta".
> *   `{item1,item2}`: Permite definir múltiples atajos o comandos en una sola línea. Por ejemplo, `super + shift + {q,r}` significa `super + shift + q` y `super + shift + r`.
> *   `bspc`: Es la herramienta de línea de comandos para interactuar con `bspwm`. Permite consultar y manipular el estado del gestor de ventanas, nodos (ventanas) y escritorios.
> *   `bspc node -f {west,south,north,east}`: Mueve el foco a la ventana adyacente en la dirección especificada.
> *   `bspc node -s {west,south,north,east}`: Intercambia la ventana actual con la ventana adyacente.
> *   `bspc node -t {tiled,pseudo_tiled,floating,fullscreen}`: Cambia el estado de la ventana (mosaico, pseudo-mosaico, flotante, pantalla completa).
> *   `bspc desktop -f {prev,next}.local`: Cambia al escritorio anterior o siguiente en el monitor actual.
> *   `bspc node -p {west,south,north,east}`: Preselecciona una dirección para la próxima ventana que se abra, útil para organizar el diseño.
> *   `pkill -USR1 -x sxhkd`: Envía la señal `USR1` al proceso `sxhkd`, lo que le indica que recargue su archivo de configuración.

> [!warning] Ruta del script de redimensionamiento
> La ruta `/home/stark/.config/bspwm/scripts/bspwm_resize` en el archivo `sxhkdrc` es un ejemplo. Asegúrese de reemplazar `/home/stark` con la ruta absoluta a su directorio `home` (por ejemplo, `/home/su_usuario/` o simplemente `~`) si el script no se encuentra en la ubicación especificada para el usuario `stark`.

## Script de redimensionamiento bspwm_resize

A continuación, se presenta el contenido del archivo de configuración `bspwm_resize` que se utiliza para el redimensionamiento de ventanas:

```bash
#!/usr/bin/env dash

if bspc query -N -n focused.floating > /dev/null; then
	step=20
else
	step=100
fi

case "$1" in
	west) dir=right; falldir=left; x="-$step"; y=0;;
	east) dir=right; falldir=left; x="$step"; y=0;;
	north) dir=top; falldir=bottom; x=0; y="-$step";;
	south) dir=top; falldir=bottom; x=0; y="$step";;
esac

bspc node -z "$dir" "$x" "$y" || bspc node -z "$falldir" "$x" "$y"
```

> [!info] Análisis del script bspwm_resize
> Este script `dash` (un shell POSIX más ligero que Bash) permite redimensionar ventanas de `bspwm` de forma dinámica.
> *   `#!/usr/bin/env dash`: Shebang que especifica el intérprete del script.
> *   `if bspc query -N -n focused.floating > /dev/null; then`: Comprueba si la ventana enfocada actualmente es flotante. `bspc query -N -n focused.floating` devuelve el ID del nodo si es flotante. `> /dev/null` descarta la salida estándar, y el éxito o fracaso del comando determina la condición.
> *   `step=20` o `step=100`: Define el tamaño del paso de redimensionamiento. Si la ventana es flotante, el paso es menor (20 píxeles) para un control más fino; si es en mosaico, el paso es mayor (100 píxeles) para cambios más rápidos.
> *   `case "$1" in ... esac`: Procesa el primer argumento pasado al script (ej. `west`, `east`, `north`, `south`).
>     *   `dir`: La dirección principal para redimensionar (ej. `right` para `west` significa expandir hacia la derecha, lo que mueve el borde izquierdo de la ventana).
>     *   `falldir`: La dirección de "fallback" si el redimensionamiento en `dir` falla (ej. si no hay espacio).
>     *   `x`, `y`: Los valores de desplazamiento en píxeles para el redimensionamiento.
> *   `bspc node -z "$dir" "$x" "$y" || bspc node -z "$falldir" "$x" "$y"`: Este es el comando clave. `bspc node -z` redimensiona la ventana enfocada. Si el redimensionamiento en la dirección principal (`$dir`) falla (por ejemplo, si se alcanza el límite de la pantalla o de otra ventana), el operador `||` (OR lógico) ejecuta el comando de redimensionamiento con la dirección de fallback (`$falldir`).

Este archivo debe ser creado en la ruta `~/.config/bspwm/scripts/` y su contenido es el mostrado anteriormente.

> [!warning] Permisos de ejecución del script
> Después de crear el archivo `bspwm_resize`, es crucial otorgarle permisos de ejecución. Esto se hace con el comando:
> ```bash
> chmod +x ~/.config/bspwm/scripts/bspwm_resize
> ```
> Sin estos permisos, `sxhkd` no podrá ejecutar el script cuando se active el atajo de teclado correspondiente.