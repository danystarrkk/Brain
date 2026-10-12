---
tags:
  - sistemas
  - entorno
  - polybar-picom
tipo: concepto
estado: terminado
---

# Instalación de Polybar y Picom

Instalación de las dos piezas visuales de la sesión de [[Configuración de BSPWM y SXHKD|bspwm]]: **polybar** desde los repositorios de la distribución y **picom** compilado desde código fuente (para tener el backend GLX y el blur `dual_kawase` que [[Configuración de Picom|la config]] exige). Al final, el cierre de sesión que aplica toda la pila.

## Instalación de Polybar

```bash
sudo apt install polybar
```

> [!info] Polybar: barra de estado modular
> Barra altamente personalizable para WMs *tiling*: módulos internos de CPU/RAM/fecha/red/volumen/batería/mpd, módulos scripteados por el usuario y bandeja XEmbed. Config en `~/.config/polybar/`. En esta máquina se usa el paquete de Debian junto con el tema y las customizaciones de [[Configuración de Polybar]].

> [!tip] Versión del paquete vs compilación
> Debian estabiliza: polybar de `apt` puede ir una o dos versiones por detrás de master (los `include-file =`, ciertos `internal/*` o tags nuevos llegan tarde). Si la config usa features de una versión reciente, hay tres caminos: compilar con CMake (ver "Compilar polybar desde fuente" abajo), usar backports (`sudo apt install -t bookworm-backports polybar`) o limitarse a las features del paquete. Verificar la versión instalada: `polybar --version`.

### Compilar polybar desde fuente (alternativa)

```bash
sudo apt install cmake libxcb1-dev libxcb-ewmh-dev libxcb-randr0-dev \
  libxcb-util-dev libxcb-icccm4-dev libxcb-image0-dev libxcb-xkb-dev \
  libxcb-xrm-dev libasound2-dev libjsoncpp-dev libmpdclient-dev \
  libcurl4-openssl-dev libuv1-dev libpixman-1-dev libcairo2-dev \
  libx11-xcb-dev cmake-data git build-essential python3-sphinx

git clone --recursive https://github.com/polybar/polybar.git
cd polybar
mkdir build && cd build
cmake .. -DCMAKE_BUILD_TYPE=Release -DCMAKE_INSTALL_PREFIX=/usr/local
make -j$(nproc)
sudo make install     # o: sudo checkinstall para generar un .deb
```

| Paso / flag | Por qué |
|---|---|
| `--recursive` | Clona también los submódulos (las librerías embebidas como `libspng`) — sin ellos el CMake falla con "submodule missing" |
| `-DCMAKE_BUILD_TYPE=Release` | Optimizaciones de compilación, sin asserts de debug |
| `-DCMAKE_INSTALL_PREFIX=/usr/local` | Prefijo estándar fuera de `/usr` — coexiste con el paquete de apt sin pisarlo |
| `make -j$(nproc)` | Compila con tantos hilos como núcleos |
| `checkinstall` en vez de `make install` | Empaqueta la instalación como `.deb` — se puede desinstalar limpiamente con apt |

> [!warning] `polybar --version` tras compilar
> Si conviven el paquete (`/usr/bin/polybar`) y el compilado (`/usr/local/bin/polybar`), el `PATH` decide cuál corre (el de `/usr/local/bin` gana si va primero — ver el PATH personal en [[Configuración Zsh Batcat Lsd y Burp Suite]]).

### Decidir entre paquete y fuente

| Criterio | `apt install polybar` | Compilar (CMake) |
|---|---|---|
| Actualizaciones | Gestor de paquetes, sin tocar nada | `git pull` + recompilar |
| Versión | La congelada de Debian (posiblemente 1–2 majors por detrás) | master / tag que elijas |
| Features nuevas de config | Pueden faltar (`include-file`, módulos recientes, tags) | Todas |
| Esfuerzo | Un comando | ~10 dependencias + 5–10 min de build |
| Desinstalación | `apt remove` | Manual, o `checkinstall` para generar `.deb` |

Regla práctica: **empezar con el paquete** (esta máquina lo usa así) y compilar solo cuando una feature concreta de la config lo exija.

## Instalación de Picom (desde código fuente)

Picom se compila para asegurar el **backend GLX** y el **blur dual_kawase** completos:

```bash
sudo apt install libconfig-dev libdbus-1-dev libegl-dev libev-dev libgl-dev libepoxy-dev libpcre2-dev libpixman-1-dev libx11-xcb-dev libxcb1-dev libxcb-composite0-dev libxcb-damage0-dev libxcb-dpms0-dev libxcb-glx0-dev libxcb-image0-dev libxcb-present-dev libxcb-randr0-dev libxcb-render0-dev libxcb-render-util0-dev libxcb-shape0-dev libxcb-util-dev libxcb-xfixes0-dev libxext-dev meson ninja-build uthash-dev

git clone https://github.com/yshui/picom
cd picom

meson setup --buildtype=release build
ninja -C build
ninja -C build install
```

> [!info] Picom: compositor ligero para Xorg
> Fork de Compton enfocado en rendimiento: elimina tearing (vsync), añade sombras, fading, transparencia y blur. Es esencial en WMs minimalistas que **no traen compositor propio** — bspwm no dibuja ni compone nada (ver [[Configuración de BSPWM y SXHKD]]). Las dependencias `lib*-dev` son los headers con los que picom habla cada subsistema de X: XCB (daño, composición, present, xfixes, shape = ventanas), GL/EGL/Epoxy (backend acelerado), pixman (raster 2D del backend xrender), libconfig (su formato de config), pcre2 (el lenguaje de condiciones de `*-exclude`), ev (bucle de eventos), dbus (API de control opcional).

> [!tip] Meson y Ninja: sistemas de construcción modernos
> **Meson** es un sistema de construcción declarativo (como CMake): lee sus flags y **genera** los archivos de build para un backend. **Ninja** es ese backend: un make de bajo nivel, ultrarrápido, nunca escrito a mano. Meson/Ninja reemplazan al autotools (`./configure && make`) y son el estándar actual de GNOME, systemd y picom. equivalencias:

| Autotools / Make | Meson + Ninja |
|---|---|
| `./configure --prefix=/usr` | `meson setup --prefix=/usr build` |
| `make` | `ninja -C build` |
| `make install` | `ninja -C build install` |
| `make clean` | `ninja -C build clean` |

### Flags de meson que conviene conocer

| Flag | Valores | Efecto |
|---|---|---|
| `--buildtype=release` | release/debug/plain/custom | Optimizaciones + `NDEBUG` |
| `-Dbackends=glx,xrender` | lista | Qué backends compilar — acota binario y dependencias |
| `-Dwith_tests=false` | bool | Omite tests (requieren gtest) |
| `-Dcompositor=false` | bool | **Peligrante**: desactiva la composición misma (solo para builds especializadas) |
| `--prefix=/usr/local` | ruta | Prefijo de instalación (defecto: `/usr/local`) |

El resultado se instala en `/usr/local/bin/picom` con sus archivos de soporte.

> [!info] Dependencias *vs* paquete
> El paquete `picom` de Debian también existe (`sudo apt install picom`), pero compilar garantiza la versión con todas las opciones del archivo de config (especialmente `corner-radius` y `blur-method = "dual_kawase"` de [[Configuración de Picom]], que llegaron tarde a stable).

### Errores típicos de compilación

| Mensaje | Causa | Fix |
|---|---|---|
| `ERROR: Dependency "libpcre2" not found` | Falta el dev de pcre2 (u otra lib) | Instalar el `*-dev` correspondiente de la lista |
| `ninja: command not found` | Falta ninja-build | `sudo apt install ninja-build` |
| `Permission denied` en `ninja -C build install` | Instalación a `/usr/local` requiere root | `sudo ninja -C build install` |
| `Failed to clone submodules` (polybar) | Falta `--recursive` | `git submodule update --init --recursive` |
| picom compila pero no arranca con blur | Backend equivocado | Verificar config `backend = "glx"` y drivers GL instalados |

### Mapa dependencias → función de picom

No todas las librerías de la lista son igual de críticas — entender el mapa ayuda a depurar builds mínimos:

| Dependencia | Habilita |
|---|---|
| `libxcb1` + `libx11-xcb-dev` | Núcleo: conexión al servidor X |
| `libxcb-damage0`, `libxcb-composite0`, `libxcb-present-dev`, `libxcb-xfixes0` | **La composición misma**: redirección de ventanas, avisos de daño, presentación, máscaras |
| `libgl-dev`, `libegl-dev`, `libepoxy-dev` | Backend **glx/egl** (blur dual_kawase, vsync eficiente) |
| `libpixman-1-dev` | Backend **xrender** (raster 2D por CPU) |
| `libpcre2-dev` | Motor de **condiciones**: todos los `shadow-exclude`, `opacity-rule`, `wintypes` |
| `libconfig-dev` | El parser de `picom.conf` |
| `libdbus-1-dev` | API D-Bus (control externo: `picom-trans`) |
| `libev-dev` | Bucle de eventos del daemon |
| `libxcb-randr0`, `libxcb-glx0`, `libxcb-image0`, `libxcb-shape0`, `libxcb-util-dev`, `libxcb-render-*`, `libxcb-dpms0` | Extensiones X concretas: multi-monitor, GLX interop, shaped/ARGB windows, utilidades, render, DPMS |
| `libxext-dev`, `uthash-dev` | Extensiones X genéricas / cabeceras hash usadas por el código |

> [!tip] Build mínimo posible
> Si solo se quiere xrender sin blur: basta `libxcb1-dev libxcb-composite0-dev libxcb-damage0-dev libxcb-image0-dev libxcb-render-util0-dev libxcb-xfixes0-dev libpixman-1-dev libconfig-dev libpcre2-dev meson ninja-build`. Quitar `libgl-dev`/`libepoxy-dev` y restringir backends elimina la necesidad del stack GL — útil en VMs sin GPU.
| picom compila pero no arranca con blur | Backend equivocado | Verificar config `backend = "glx"` y drivers GL instalados |

## Verificación y desinstalación

Tras el cierre de sesión (siguiente sección), checklist de verificación:

```bash
polybar --version; picom --version        # versiones efectivas (¿paquete o /usr/local?)
pgrep -a picom; pgrep -a polybar          # procesos vivos
polybar -m                                 # monitores vistos por polybar
picom --config ~/.config/picom/picom.conf  # foreground: errores visibles
```

Desinstalación ordenada:

| Componente | Paquete | Compilado |
|---|---|---|
| polybar | `sudo apt remove polybar` | borrar `/usr/local/bin/polybar` y `/usr/local/share/polybar` (o el `.deb` de checkinstall) |
| picom | `sudo apt remove picom` | borrar `/usr/local/bin/picom` + datos en `/usr/local/share/picom` |

Y quitar sus líneas de `bspwmrc` si la sesión debe volver a arrancar sin ellas.

## Autostart: cómo quedan integrados

Ninguno de los dos necesita servicio systemd — son procesos de usuario arrancados por el WM. En `~/.config/bspwm/bspwmrc`:

```bash
picom &                       # compositor, ver [[Configuración de Picom]]
~/.config/polybar/launch.sh & # barras, ver [[Configuración de Polybar]]
```

| Mecanismo | Cuándo aplica | Ejemplo |
|---|---|---|
| `bspwmrc` (elegido aquí) | Sesiones X con bspwm | `picom &` |
| `xprofile` / `.xinitrc` | Arranque X genérico | Lo mismo, antes de `exec bspwm` |
| Servicio systemd de usuario | Escritorios con systemd --user | `systemctl --user enable picom.service` con un unit file custom — innecesario con bspwm |

El `launch.sh` de polybar mata instancias viejas, espera a que mueran y relanza las barras (mecánica completa en [[Configuración de Polybar]]); el tema [[Instalación y Configuración de Blue Sky Polybar]] trae su propia versión de ese lanzador.

## Aplicar: cierre de sesión

Para que toda la pila (WM, compositor, barras, shell) arranque con la nueva instalación:

```bash
pkill -x bspwm
```

o desde una sesión gráfica: cerrar la sesión del WM — `bspc quit` o `super + shift + q` con la config personal de [[Configuración de BSPWM y SXHKD]] — y volver a iniciarla desde el login. Al volver, `bspwmrc` lanzará picom y polybar ya instalados, con toda su configuración.

> [!tip] Verificación tras re-login
> - `pgrep -a picom` → proceso corriendo.
> - `polybar -m` → monitores listados; `pgrep -a polybar` → instancias vivas.
> - Si picom falla, correrlo en foreground para leer el error: `picom --config ~/.config/picom/picom.conf`.

## Relacionadas

- [[Configuración de Picom]] — el `picom.conf` completo y sus opciones
- [[Configuración de Polybar]] — los `.ini` y scripts de las barras
- [[Instalación y Configuración de Blue Sky Polybar]] — el tema base de polybar
- [[Configuración de BSPWM y SXHKD]] — el WM y su `bspwmrc`
- [[Configuración Post-Instalación de Entorno de Escritorio]] — el resto del checklist
- [[Bash Scripting - Fundamentos Básicos]] — base para leer/modificar `launch.sh` y `bspwmrc`
