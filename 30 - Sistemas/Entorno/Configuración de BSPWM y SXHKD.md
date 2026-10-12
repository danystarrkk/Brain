---
tags:
  - sistemas
  - entorno
  - sxhkd
tipo: concepto
estado: terminado
---

# Configuración de BSPWM y SXHKD

**BSPWM** (*Binary Space Partitioning Window Manager*) es un gestor de ventanas de mosaico para X11 que modela tus ventanas como las **hojas de un árbol binario**. **SXHKD** (*Simple X Hotkey Daemon*) es el demonio de atajos de teclado que lo controla: bspwm **no lee ningún archivo de configuración propio** — todo (focos, movimientos, estados) se define ejecutando el cliente `bspc`, y sxhkd traduce teclas en comandos. Esa separación total entre "quién gestiona" y "quién escucha teclas" es la que hace a bspwm minimalista y scripteable.

> [!info] El modelo mental: bspwm no dibuja nada
> A diferencia de GNOME o de un WM " clásico", bspwm no dibuja bordes decorativos, no tiene barras propias ni menús. Solo hace tres cosas: recibe eventos de X11, mantiene el árbol de nodos y coloca/mueve ventanas. Todo lo visible (barra → [[Configuración de Polybar|Polybar]], sombras → [[Configuración de Picom|picom]], lanzadores → rofi) son procesos externos que bspwm solo *llama* desde su `bspwmrc`.

## Cómo funciona un tiling WM en X11

Para entender la configuración hay que entender la arquitectura subyacente:

| Componente | Rol |
|---|---|
| **Servidor X (Xorg)** | Dueño de la pantalla y del input. Apila ventanas por orden de creación/Stacking |
| **WM de tipo stacking/flotante** (Openbox, Windows, macOS) | El servidor X permite que cada ventana pida su posición y tamaño; el WM solo "apila" y gestiona foco |
| **WM de mosaico** (bspwm, i3, dwm) | El WM **ignora** las peticiones de geometría y asigna posición/tamaño él mismo, dividiendo el espacio en mosaicos |
| **XCB** (X C Binding) | API de comunicación cliente–servidor X, más eficiente que la antigua Xlib. bspwm está escrito sobre ella — de ahí las dependencias `libxcb-*` |

Los WMs de mosaico se dividen en dos familias según **quién define el layout**:

- **Manuales** (i3, sway): el usuario decide explícitamente "split horizontal, split vertical" con atajos.
- **Automáticos** (bspwm, dwm, herbstluftwm): el WM decide dónde inserta cada ventana nueva, y el usuario navega/ajusta el resultado. bspwm lo hace con su árbol binario.

> [!tip] ¿Por qué bspwm y no i3?
> i3 trae integrado todo (barras, layout de workspaces, configuración declarativa). bspwm trae solo el kernel del mosaico y te obliga a componer: sxhkd para teclas, polybar para la barra, picom para el compositor. Es más piezas que armar, pero cada pieza es reemplazable y todo es scripteable vía `bspc`. Ver [[Instalación de Polybar y Picom]] y [[Instalación y Configuración de Blue Sky Polybar]].

## El modelo de árbol de bspwm

bspwm organiza la jerarquía así:

```
monitor (DP-1, eDP-1…)
└── desktop / workspace (I, II, III…)
    └── nodo interno (partición)
        ├── nodo interno
        │   ├── ventana (hoja)
        │   └── ventana (hoja)
        └── ventana (hoja)
```

| Concepto | `bspc` lo llama | Significado |
|---|---|---|
| **Monitor** | `monitor` | Una salida física (xrandr). Cada monitor tiene sus escritorios |
| **Escritorio** | `desktop` | Un "workspace". Contiene un árbol de nodos |
| **Nodo interno** | `internal node` | Una partición del espacio: tiene dos hijos y una dirección del split (horizontal/vertical) |
| **Ventana** | `leaf node` / `window` | Una hoja del árbol: la ventana real de X11 |
| **Preselección** | `preselection` | Marcador temporal: indica dónde irá la *próxima* ventana abierta |

Cada ventana nueva se inserta como hermana del árbol (`brother`, en bspwm el hijo secundario) del nodo enfocado, y el split se elige según la geometría: si la ventana entra "al lado", split horizontal; si "debajo", vertical. Con preselección (`bspc node -p west` etc.) fuerzas la inserción manualmente.

Estados de una ventana (la `t` de `super + t` de la config personal):

| Estado | Comportamiento |
|---|---|
| `tiled` | Mosaico normal: ocupa su caja del árbol |
| `pseudo_tiled` | Respeta su tamaño *preferido* pero nunca sale de su caja (si no cabe, se comprime) |
| `floating` | Fuera del árbol de mosaico: posición/tamaño libres |
| `fullscreen` | Ocupa todo el monitor, oculta bordes y barra si el WM lo permite |

Flags adicionales (`super + ctrl + {m,x,y,z}` de la config personal): `marked` (destacado para moverlo luego con `newest.marked`), `locked` (ignora `--follow` al moverte de escritorio), `sticky` (visible en todos los escritorios), `private` (transparencia atenuada, marca visual de "no tocar").

## Instalación desde código fuente

### Dependencias de compilación

```bash
sudo apt install build-essential git vim xcb libxcb-util0-dev libxcb-ewmh-dev libxcb-randr0-dev libxcb-icccm4-dev libxcb-keysyms1-dev libxcb-xinerama0-dev libasound2-dev libxcb-xtest0-dev libxcb-shape0-dev
```

> [!info] Qué instala cada bloque
> - `build-essential`: `gcc`, `g++`, `make` — la colección mínima para compilar C en Debian/Ubuntu.
> - `git`: clonar los repos; `vim`: editar los configs (o el editor que prefieras, ver [[Vim Comandos Esenciales]]).
> - `libxcb-*-dev`: implementación de cada extensión XCB que bspwm/sxhkd usan: `ewmh` (estándar de escritorios virtuales), `randr` (multi-monitor), `icccm` (protocolo de gestión de ventanas), `keysyms` (mapeo teclas), `xinerama` (pantallas múltiples), `xtest` (inyección de eventos, la usa sxhkd), `shape` (formas de ventana).
> - `libasound2-dev`: soporte de audio (ALSA) en sxhkd para efectos de sonido opcionales.
>
> El paquete `xcb` "pelado" de la lista es un meta-paquete simbólico: las librerías reales son las `libxcb-*-dev`.

Actualizamos el índice y clonamos los dos proyectos:

```bash
apt update
cd ~/Descargas
git clone https://github.com/baskerville/bspwm.git
git clone https://github.com/baskerville/sxhkd.git
```

> [!tip] `git clone`
> Descarga una copia completa del repositorio remoto (código + historial) a un directorio local. `bspwm` es el WM; `sxhkd` el demonio de hotkeys. Ambos se compilan por separado: entran a cada directorio y ejecutan `make` y `sudo make install`.
>
> `make` compila usando el `Makefile` del proyecto; `sudo make install` copia los binarios a `/usr/local/bin` y los archivos de datos a `/usr/local/share`, requiriendo root.

```bash
cd bspwm && make && sudo make install && cd ..
cd sxhkd && make && sudo make install && cd ..
```

> [!tip] Compilar en dos pasos
> `make` (sin root) compila en el directorio local — si falla por falta de dependencias, no has ensuciado nada del sistema. `make install` recién copia los artefactos. Un `make clean` entre builds evita artefactos stalé cuando actualices con `git pull`.

### Estructura de configuración

```bash
mkdir -p ~/.config/{bspwm,sxhkd}
cp ~/Descargas/bspwm/examples/bspwmrc ~/.config/bspwm/
cp ~/Descargas/bspwm/examples/sxhkdrc ~/.config/sxhkd/
```

- `~/.config/` es la ubicación estándar XDG para configuración de usuario.
- `bspwmrc` es el script que bspwm ejecuta al arrancar (y con `bspc wm -r` al reiniciar): aquí se lanzan sxhkd, picom, polybar, feh… Ver "Autostart" más abajo.
- `sxhkdrc` define los atajos.

No olvidar hacerlos ejecutables: `chmod +x ~/.config/bspwm/bspwmrc` — sin ese bit, la sesión inicia "en crudo" (sin sxhkd ni nada).

### Paquetes complementarios

```bash
sudo apt install kitty rofi imagemagick feh wmname
```

| Paquete | Rol en el entorno |
|---|---|
| `kitty` | Terminal acelerado por GPU; abre con `super + Return` |
| `rofi` | Lanzador de apps/selectores (reemplazo moderno de dmenu); abre con `super + d` |
| `imagenagick`/`imagemagick` | Capturas de pantalla y manipulación de imágenes |
| `feh` | Fondo de escritorio (`feh --bg-fill`) e imágenes |
| `wmname` | Fija el nombre del WM reportado a las apps Java (necesario para Burp Suite, ver [[Configuración Zsh Batcat Lsd y Burp Suite]]) |

## `sxhkdrc`: sintaxis del demonio de atajos

sxhkd tiene una sintaxis propia de dos líneas por binding: la **chord** (teclas) y el **comando** (con sangría):

```
# comentario = descripción del binding
super + {_,shift + }{Left,Down,Up,Right}
	bspc node -{f,s} {west,south,north,east}
```

| Elemento | Sintaxis | Ejemplo | Expansión |
|---|---|---|---|
| Modificadores | `super`, `alt`, `ctrl`, `shift` | `super + shift + q` | — |
| Comodín de chord | `{a,b,c}` | `super + {t,s,f}` | 3 chords: `super+t`, `super+s`, `super+f` |
| Modificador opcional | `_` dentro del grupo | `super + {_,shift + }q` | `super+q` **y** `super+shift+q` |
| Rango de teclas | `{1-9,0}` | `super + {1-9,0}` | `super+1` … `super+9`, `super+0` |
| Comando multi-línea | `\` al final | `bspc wm -h off; \` | Las líneas siguientes (con sangría) forman un solo comando |
| Repetición | mantener pulsado | `super + j` sostenido | sxhkd re-firea mientras la chord esté activa |

Reglas importantes:

- La chord usa **todos** los modificadores de forma literal: `super + shift + Left` solo dispara si *exactamente* Super+Shift+Left están pulsados (más teclas rompen el match).
- El comando se ejecuta vía `/bin/sh -c`; `$` hay que escaparlo como `\$` si es literal.
- `pkill -USR1 -x sxhkd` recarga la config en caliente: la señal `USR1` le indica a sxhkd releer su `sxhkdrc` sin reiniciar el demonio (`-x` = match exacto del nombre de proceso).

## El `sxhkdrc` personal, anotado

Esta es la configuración real de uso diario. Cada bloque, con su mecánica:

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
```

Los tres bindings "de sistema": terminal (`kitty`), lanzador de apps (`rofi -show run` muestra y ejecuta binarios del `PATH`) y recarga de sxhkd sin tocar la sesión.

```bash
#
# bspwm hotkeys
#

# quit/restart bspwm
super + shift + {q,r}
	bspc {quit,wm -r}

# close and kill
super + {_,shift + }q
	bspc node -{c,k}
```

| Binding | Comando | Acción |
|---|---|---|
| `super + shift + q` | `bspc quit` | Cierra bspwm (termina la sesión X) |
| `super + shift + r` | `bspc wm -r` | **Reinicia** bspwm re-ejecutando `bspwmrc` — el ciclo de configuración es editar → `super+shift+r` |
| `super + q` | `bspc node -c` | Cierra la ventana con el protocolo WM_DELETE (cierre limpio: la app puede preguntar "¿guardar?") |
| `super + shift + q` | `bspc node -k` | **Kill**: cierra la ventana ignorando el protocolo (mata apps colgadas) |

```bash
# alternate between the tiled and monocle layout
super + m
	bspc desktop -l next

# send the newest marked node to the newest preselected node
super + y
	bspc node newest.marked.local -n newest.!automatic.local

# swap the current node and the biggest window
super + g
	bspc node -s biggest.window
```

- `-l next` rota los layouts del escritorio: `tiled` ↔ `monocle` (una ventana ocupa todo, tipo maximizado).
- `super + y` es el "mover marcado a preseleccionado": marcas una ventana con `super+ctrl+m` (`-g marked`), preseleccionas destino con `super+ctrl+alt+flecha`, y la envías.
- `biggest.window` es un **descriptor** de bspwm (ver catálogo abajo): selecciona la ventana más grande.

```bash
#
# state/flags
#

# set the window state
super + {t,shift + t,s,f}
	bspc node -t {tiled,pseudo_tiled,floating,fullscreen}

# set the node flags
super + ctrl + {m,x,y,z}
	bspc node -g {marked,locked,sticky,private}
```

Los estados y flags descritos en el modelo de árbol arriba. `super + shift + t` deja flotante la ventana actual; `super + s` alterna flotante↔mosaico.

```bash
# focus/swap

# focus the node in the given direction
super + {_,shift + }{Left,Down,Up,Right}
	bspc node -{f,s} {west,south,north,east}

# focus the node for the given path jump
super + {p,b,comma,period}
	bspc node -f @{parent,brother,first,second}
```

La navegación direccional usa la **geometría** (`west` = ventana que física/quemente está a la izquierda), mientras que `@{...}` navega por el **árbol**: `parent` = la partición que contiene la ventana actual, `brother` = la otra rama de esa partición, `first`/`second` = primer/segundo hijo. Aprender `@` es la clave para no perderse con muchas ventanas.

```bash
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
```

Puntos técnicos destacables:

- `.{next,prev}.local.!hidden.window` es **lenguaje de selectores** de bspwm: `local` = dentro del escritorio actual; `!hidden` = excluye ventanas minimizadas/ocultas; `.window` = el match debe ser una hoja-ventana.
- `super + grave` (backtick) y `super + Tab` saltan al último nodo/escritorio usado — el "alt-tab" del tiling.
- El bloque `older/newer` suspende el historial de foco (`wm -h off`), salta, y lo reactiva (`wm -h on`) — sin eso, el propio salto enredaría el historial.
- `'^{1-9,10}'`: el `^` convierte el número en **índice absoluto** de escritorio; nota que `0` se mapea al escritorio 10 con `^{10}`.

```bash
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
```

Flujo de preselección: `super+ctrl+alt+flecha` marca "la próxima ventana va a la izquierda/abajo/…", `super+ctrl+N` ajusta el ratio de la partición (`-o 0.3` = 30%), y se cancela con uno de los dos `space`. El último binding es un mini-pipe: `bspc query -N -d` lista los nodos del escritorio y `xargs` les cancela la preselección uno a uno.

```bash
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

`-v` es un **vector de desplazamiento** en píxeles: `{-20 0,0 20,0 -20,20 0}` son los cuatro pares (x,y) para west/south/north/east. El resize roto (izquierda/abajo/arriba/derecha) se delega al script propio `bspwm_resize` — ver la siguiente sección.

> [!warning] Ruta absoluta del script
> El binding usa la ruta absoluta `/home/stark/.config/bspwm/scripts/bspwm_resize`. En otra máquina habrá que ajustarla al `home` propio (o usar `$HOME`, que sxhkd expande vía `sh -c`).

## Script `bspwm_resize`: redimensionar con fallback

Creado en `~/.config/bspwm/scripts/bspwm_resize` con este contenido:

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

Mecánica, línea a línea:

| Línea | Qué hace |
|---|---|
| `#!/usr/bin/env dash` | Shebang: corre con `dash` (shell POSIX más ligero que bash; elige `dash` porque no usa features de bash) |
| `bspc query -N -n focused.floating` | Devuelve el ID del nodo **si** la ventana enfocada cumple el descriptor `focused.floating`. El código de salida decide el `if`: flotante → paso fino (20 px), mosaico → paso grueso (100 px) |
| `case "$1" in ...` | El argumento (`west/east/north/south` desde sxhkd) define la dirección |
| `dir` / `falldir` | `bspc node -z` redimensiona **estirando el borde opuesto**: para `west` (mover el borde izquierdo) el borde que trabaja es `right` con x negativo; el `falldir` es el borde espejo por si no hay espacio |
| `\|\| bspc node -z "$falldir" ...` | El operador OR: si el primer `-z` falla (límite de pantalla u otra ventana), redimensiona desde el borde opuesto con el mismo desplazamiento |

> [!warning] Permisos de ejecución
> `chmod +x ~/.config/bspwm/scripts/bspwm_resize` es obligatorio: sin el bit de ejecución, sxhkd intenta lanzarlo y falla en silencio (el comando `sh -c` devuelve error, sxhkd no avisa). Si un binding "no hace nada", revisa permisos y rutas antes que la chord.

## Autostart: xinitrc, bspwmrc y el orden de arranque

La cadena de arranque de una sesión X con startx:

```
login → startx → ~/.xinitrc → (ejecuta bspwm en foreground) → bspwm lee ~/.config/bspwm/bspwmrc
```

1. **`~/.xinitrc`**: último renglón debe ser `exec bspwm` (el `exec` reemplaza el shell por bspwm, sin dejar un proceso padre colgando).
2. **`bspwmrc`**: script ejecutado por bspwm al inicio y en cada `bspc wm -r`. Patrón canónico:

```bash
#!/bin/sh

# compositing y barra
picom &                       # ver [[Configuración de Picom]]
~/.config/polybar/launch.sh & # ver [[Configuración de Polybar]]

# teclado, fondo, apps al arrancar
sxhkd &
feh --bg-fill ~/wallpaper.jpg
wmname LG3D                   # solo si usas apps Java/Burp, ver [[Configuración Zsh Batcat Lsd y Burp Suite]]
```

> [!warning] El `&` importa
> `bspwmrc` se ejecuta en serie: un proceso sin `&` que nunca termina (picom, sxhkd, polybar) **bloquearía el resto del script** y bspwm quedaría a medio arrancar. Los procesos daemon van con `&`; los comandos de una sola vuelta (`feh`, `wmname`, `bspc config ...`) pueden ir en serie.

> [!tip] Ciclo de configuración
> Tras editar `sxhkdrc`: `super + Escape` recarga sxhkd al instante. Tras editar `bspwmrc` o configs de bspwm: `super + shift + r` (`bspc wm -r`) reinicia el WM sin cerrar la sesión — las apps abiertas sobreviven.

## Catálogo de comandos `bspc`

`bspc` es la API completa del WM. Estructura: `bspc <objeto> -<flag> [selector]`.

### Objetos

| Objeto | Descripción |
|---|---|
| `node` | Ventanas y particiones (lo más usado) |
| `desktop` | Escritorios/workspaces |
| `monitor` | Salidas físicas |
| `wm` | El WM mismo (historial, reglas, monitores) |
| `query` | Consultas (lectura del estado, base de los scripts) |
| `config` | Lee/escribe `bspwmrc`-settings en caliente (`bspc config border_width 2`) |
| `rule` | Reglas de inserción: `bspc rule -a Firefox desk=^2` |

### Flags de `bspc node` (los que usa la config + extras)

| Flag | Función | Ejemplo |
|---|---|---|
| `-f` | Mover el **foco** al match | `bspc node -f west` |
| `-s` | **Swap** con el match | `bspc node -s biggest.window` |
| `-d` | Mover la ventana al escritorio | `bspc node -d '^2'` |
| `-t` | Cambiar estado | `bspc node -t fullscreen` |
| `-g` | Cambiar flags | `bspc node -g sticky` |
| `-v` | Mover (vector) | `bspc node -v 20 0` |
| `-z` | Redimensionar (borde+deltas) | `bspc node -z right 100 0` |
| `-p` | Preselección | `bspc node -p cancel` |
| `-o` | Ratio de la preselección | `bspc node -o 0.6` |
| `-n` | Renombrar/re-insertar | `bspc node -n chat` |
| `-r` | Rotar la partición | `bspc node -r 90` |
| `-B` / `-E` | Balancear / estirar hojas | `bspc node @/ -B` |

### Descriptores y selectores (el lenguaje de match)

| Selector | Ejemplo | Significado |
|---|---|---|
| Direcciones | `west`, `south`, `north`, `east` | Vecino geométrico |
| Rutas de árbol | `@/`, `@parent`, `@brother`, `@first`, `@second` | Navegación estructural |
| Cadenas de focus | `.local.!hidden.window` | Scoped al escritorio, no ocultas, tipo ventana |
| Especiales | `biggest.window`, `newest.window`, `focused.window`, `!focused.window` | Predefinidos |
| Marcas | `newest.marked.local` | Nodos con flag `marked` |
| Índices | `'^{1-9}'`, `^2` | Escritorio por número |

### `query`: la base del scripting

```bash
bspc query -N -n focused.floating    # ID si la enfocada es flotante (usado en bspwm_resize)
bspc query -T                          # estado completo del WM en JSON (árboles, focos)
bspc query -D --names                  # listar escritorios
bspc query -N -d .local.window         # ventanas del escritorio actual
```

Cualquier salida de `query` se canaliza con `xargs`/pipes a otros `bspc` — es la forma de escribir scripts como `bspwm_resize` o los indicadores de [[Configuración de Polybar]].

## Errores comunes

1. **La sesión arranca "en crudo"** (sin atajos): falta `chmod +x ~/.config/bspwm/bspwmrc`, o el `bspwmrc` no lanza `sxhkd &`.
2. **`super + shift + r` y todo desaparece**: `bspwmrc` sin `&` en procesos largos, o un comando que falla y aborta el resto del script (`set -e` implícito en sh no aplica, pero un error temprano salta pasos posteriores).
3. **Un binding deja de funcionar tras editar**: ejecuta `pkill -USR1 -x sxhkd` (`super + Escape`) — sxhkd no recarga solo.
4. **`bspc: command not found` en sxhkd**: la ruta de `bspc` no está en el `PATH` de la sesión de X (usa ruta absoluta `/usr/local/bin/bspc` en el binding si ocurre).
5. **Bindings que "no hacen nada" con scripts**: permisos de ejecución ausentes o ruta absoluta incorrecta (el caso `/home/stark/...` en otra máquina).
6. **Java/Burp con ventanas diminutas o sin bordes**: falta `wmname LG3D` + `_JAVA_AWT_WM_NONREPARENTING=1`, ver [[Configuración Zsh Batcat Lsd y Burp Suite]].

## Recursos

- bspwm: <https://github.com/baskerville/bspwm> — README con el modelo de árbol y selectores.
- sxhkd: <https://github.com/baskerville/sxhkd/wiki> — wiki con la sintaxis de chords completa.
- `man bspc`, `man bspwm`, `man sxhkd` — la referencia definitiva de selectores y flags.

## Relacionadas

- [[Configuración de Picom]] — el compositor que se lanza desde `bspwmrc`
- [[Configuración de Polybar]] — la barra y sus módulos (incluye scripts personalizados)
- [[Instalación de Polybar y Picom]] — compilación de picom y arranque
- [[Instalación y Configuración de Blue Sky Polybar]] — el tema de la barra
- [[Configuración Post-Instalación de Entorno de Escritorio]] — checklist completa de pos-install
- [[Configuración de Zsh y Plugins]] — el shell dentro de kitty
- [[Comandos Básicos de Sistema y Red en Linux]] — fundamentos de procesos, señales y red
- [[Bash Scripting - Fundamentos Básicos]] — la base para escribir scripts como `bspwm_resize`
