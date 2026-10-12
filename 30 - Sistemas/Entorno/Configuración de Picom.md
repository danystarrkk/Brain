---
tags:
  - sistemas
  - entorno
  - picom
tipo: concepto
estado: terminado
---

# Configuración de Picom

**Picom** (antes *compton*) es un **compositor para X.org**: un intermediario que intercepta todo lo que las ventanas dibujan, lo mezcla en una superficie off-screen y lo presenta en pantalla de una sola vez. Ese rol le permite añadir **eliminación de tearing (vsync), sombras, fading, transparencia, blur y esquinas redondeadas** — efectos que un WM de mosaico como bspwm no provee. Es el pieza central de la estética de [[Configuración de BSPWM y SXHKD|bspwm]].

> [!info] ¿Qué hace exactamente un compositor?
> Sin compositor, cada ventana dibuja **directamente sobre la pantalla** (la "raíz" de X) y el servidor X apila los pixmaps. Con un compositor activo, X redirige (*redirect*) cada ventana a un pixmap off-screen; el compositor compone esos pixmaps en el orden correcto (z-order, transparencias, sombras) y sube el resultado final a la pantalla. Todo el dibujo pasa por un proceso intermedio — por eso picom "ve" cada ventana y puede difuminarla, sombrearla o atenuarla.

## El problema que resuelve: tearing y cómo lo ataca el vsync

El *tearing* aparece cuando la tarjeta gráfica presenta la imagen a mitad del barrido del monitor: ves la mitad superior del frame nuevo y la inferior del viejo. Picom lo ataca con **vsync**, que sincroniza el blit final con el refresco del monitor:

| Mecanismo | Cómo funciona | Tradeoff |
|---|---|---|
| **vsync (GLX/dri)** | Espera al *vertical blanking interval* antes de presentar | Latencia añadida mínima; elimina tearing por completo |
| **Doble buffer + swap** | Backend GLX usa buffers intercambiables | Requiere GL; no disponible en xrender |
| `unredir-if-possible` | Si una ventana fullscreen opaca aparece, picom **sale del medio** (sin redirección) | Máximo rendimiento en juegos; conocido provocar parpadeos al entrar/salir |

> [!warning] `vsync = false` en la config personal
> La config actual lleva `vsync = false` con `backend = "glx"`. Sin vsync puede aparecer tearing en vídeo/arrastres de ventanas. Si se ve, cambiar a `vsync = true` es el primer ajuste a probar — en GLX moderno el costo de latencia es despreciable.

## Backends de renderizado

```ini
backend = "glx";     # la elección personal
```

| Backend | Motor | Ventajas | Inconvenientes |
|---|---|---|---|
| `xrender` | Render extension de X (2D por CPU) | Máxima compatibilidad; sin requerimientos de GPU | Sin blur de calidad; peor rendimiento con muchos efectos |
| `glx` | OpenGL sobre GLX | Rendimiento alto; blur `dual_kawase`; vsync eficiente | Requiera GL funcional; en NVIDIA viejo puede necesitar `xrender-sync-fence` |
| `xr_glx_hybrid` | Intenta glx y cae a xrender | Transparencia ante fallos | Comportamiento impredecible; hoy rara vez se usa |
| `egl` (picom ≥11) | OpenGL vía EGL | Alternativa moderna, Wayland-friendly | Soporte de efectos aún más joven |

> [!tip] Regla práctica
> GPU con drivers buenos → `glx`. Todo lo demás (VMs, NVIDIA proprietary con problemas, integraciones raras) → `xrender`. El híbrido es un parche, no una elección de diseño.

## Métodos de blur

La config usa:

```ini
blur-method = "dual_kawase"
blur-size = 2
blur-strength = 3
blur-background = true
```

| Método | Algoritmo | Calidad | Coste | Notas |
|---|---|---|---|---|
| `none` | Sin blur | — | 0 | Por defecto en upstream |
| `box` | Convolución box (`blur-kern`) | Baja | Alto | El original de compton; escala mal |
| `gaussian` | Convolución gaussiana (`blur-kern`) | Media-alta | Muy alto | Kerns grandes son carísimos por CPU |
| `kawase` | Downsample + blur por pasos (Kawase) | Buena | Medio | Implementación de ibhagwan/fork |
| `dual_kawase` | Kawase en dos fases (down + upsample) | Excelente | Bajo | **La elección personal**; requiere backend GLX/EGL |

El **dual Kawase blur** difumina por aproximación escalonada en vez de por convolución: redimensiona la imagen hacia abajo, aplica passes de Kawase y la reescala — coste casi constante independiente del radio. Por eso `blur-size = 2` + `blur-strength = 3` dan un blur suave y barato, mientras un `blur-kern` gaussiano de 21×21 quemaría la CPU. `blur-strength` (1–20) solo aplica a los métodos Kawase; los métodos por kernel usan `blur-kern`.

> [!warning] Blur requiere transparencia real
> El blur solo se aplica **detrás de ventanas semitransparentes/ARGB**: una ventana opaca no deja ver nada que difuminar. Con `inactive-opacity = 1.0` (config actual) el blur solo se nota en ventanas con transparencia propia (terminales translúcidos, launchers).

## La configuración personal completa

Archivo: `~/.config/picom/picom.conf`. Se crea con:

```bash
mkdir -p ~/.config/picom
nano ~/.config/picom/picom.conf
```

> [!tip] Estructura XDG
> `~/.config/` sigue el [XDG Base Directory Spec](https://specifications.freedesktop.org/basedir-spec/basedir-latest.html); el config se puede respaldar/migrar copiando ese directorio.

### Esquinas redondeadas

```ini
corner-radius = 20;
rounded-corners-exclude = [
  #"window_type = 'normal'",
  #"class_g = 'firefox'",
];

round-borders = 20;
round-borders-exclude = [
  #"class_g = 'TelegramDesktop'",
];

round-borders-rule = [];
```

| Opción | Valores | Significado |
|---|---|---|
| `corner-radius` | píxeles | Radio de las esquinas redondeadas de todas las ventanas (20 = muy redondeado) |
| `rounded-corners-exclude` | lista de condiciones | Ventanas que quedan con esquinas rectas |
| `round-borders` | píxeles | Redondea también los bordes dibujados por el WM |
| `round-borders-rule` | `"PIXELS:PATTERN"` | Radio por ventana, ej. `"2:class_g = 'URxvt'"` |

> [!info] Origen de las esquinas redondeadas
> El comentario `requires: github.com/sdhand/compton` es histórico: `corner-radius` nació en los forks (compton-fork → picom-ibhagwan) y **fue fusionado upstream en picom v10+**. Con el paquete actual de picom (ver [[Instalación de Polybar y Picom]]) funciona sin forks.

### Sombras

```ini
shadow = true
shadow-radius = 15
shadow-opacity = .75
shadow-offset-x = -15
shadow-offset-y = -15
shadow-exclude = [
    "class_g = 'firefox' && argb"
];
```

| Opción | Valores (defecto) | Significado en esta config |
|---|---|---|
| `shadow` | bool | Sombras activas (15 px de desenfoque, 75 % de opacidad) |
| `shadow-radius` | px (12) | Radio de desenfoque del borde de la sombra |
| `shadow-opacity` | 0–1 (0.75) | Opacidad; 1.0 = sombra negra sólida |
| `shadow-offset-x/y` | px (−15/−15) | Desplazamiento: negativos = sombra abajo-izquierda |
| `shadow-exclude` | condiciones | Exclusiones. Aquí: ventanas **ARGB de Firefox** (ya llevan su propio canal alfa; sombra encima se ve sucia) |

También existen `shadow-red/green/blue` (tintar la sombra; por defecto negra) y `shadow-exclude-reg` (región X sin sombra, útil bajo docks).

> [!tip] Lenguaje de condiciones
> Todas las exclusiones (`shadow-exclude`, `opacity-rule`, `focus-exclude`…) usan el mismo lenguaje: `class_g` = clase X11 (WM_CLASS, `xprop | grep WM_CLASS` la muestra), `name` = WM_NAME, `window_type` = tipo EWMH, operadores `&&`, `||`, `!`, y matching `*=`, `*=~`, `==`. Es el mismo lenguaje que usan polybar y otras herramientas de la pila X11.

### Fading (desvanecimiento)

```ini
#fading = true
fade-in-step = 0.01;
fade-out-step = 0.01;
# fade-delta = 10
```

| Opción | Valores (defecto) | Significado |
|---|---|---|
| `fading` | bool (false) | Habilita fade al abrir/cerrar ventanas |
| `fade-in-step` | 0.01–1 (0.028) | Opacidad ganada por paso al aparecer (0.01 = fade más lento/suave que el defecto) |
| `fade-out-step` | 0.01–1 (0.03) | Opacidad perdida por paso al cerrar |
| `fade-delta` | ms (10) | Duración entre pasos |
| `no-fading-openclose` | bool | Desactiva el fade de apertura/cierre manteniendo otros |

> [!warning] Corrección técnica: `fading` está comentado
> En la config actual la línea `#fading = true` queda **comentada**, y el valor por defecto de picom es `fading = false`. Los `fade-*-step` ajustados **no harán nada** hasta descomentar `fading = true`. Si el fade deseado no aparece, esa es la causa.

### Opacidad

```ini
inactive-opacity = 1.0
frame-opacity = 1.0
opacity = 1.0
inactive-opacity-override = false;
active-opacity = 1.0
focus-exclude = [ "class_g = 'Cairo-clock'" ];
# opacity-rule = [ "98:class_g = 'Polybar'" ]
```

| Opción | Valores | Significado en esta config |
|---|---|---|
| `inactive-opacity` | 0.1–1 | **1.0**: sin atenuación de ventana inactiva (se puede bajar p. ej. a 0.9 para resaltar el foco) |
| `frame-opacity` | 0.1–1 | Opacidad de bordes/titlebars (irrelevante en bspwm sin marcos propios) |
| `opacity` | 0–1 | Opacidad de menús/dropdowns |
| `active-opacity` | 0–1 | Opacidad de la ventana activa |
| `inactive-opacity-override` | bool | false: `_NET_WM_OPACITY` de la app gana sobre `inactive-opacity` |
| `focus-exclude` | condiciones | Siempre "enfocada": aquí `Cairo-clock` (reloj que anima y no debe atenuarse) |
| `opacity-rule` | `"PCT:PATTERN"` | Opacidad por ventana; comentada: polybar al 98 % |

`inactive-dim` (0–1) es el mecanismo alternativo: atenúa **restando brillo** en vez de transparencia (más barato, no deja ver lo de atrás).

### Blur

```ini
blur-method = "dual_kawase"
blur-size = 2
blur-strength = 3
blur-background = true
```

Ya referenciado arriba: dual Kawase, tamaño mínimo, fuerza baja — blur sutil. `blur-background-frame` (blur detrás de marcos opacos) y `blur-background-fixed` quedan en sus defectos.

### General

```ini
backend = "glx";
vsync = false
mark-wmwin-focused = true;
mark-ovredir-focused = true;
detect-rounded-corners = true;
detect-client-opacity = true;
refresh-rate = 0
detect-transient = true;
detect-client-leader = true;
use-damage = false
log-level = "warn";
```

| Opción | Valores | Por qué esta config |
|---|---|---|
| `backend` | `glx`/`xrender`/`egl` | GLX para blur dual_kawase y composición eficiente |
| `vsync` | bool | **false** (ver advertencia de tearing arriba) |
| `mark-wmwin-focused` | bool | Marca como enfocadas las ventanas "de gestión" del WM — arregla fading/opacidad en ventanas especiales |
| `mark-ovredir-focused` | bool | Ídem para ventanas *override-redirect* (menus, tooltips) sin `WM_STATE` |
| `detect-rounded-corners` | bool | Evita confundir ventanas redondeadas con ventanas *shaped* |
| `detect-client-opacity` | bool | Lee `_NET_WM_OPACITY` del cliente — necesario para que apps fijen su propia transparencia |
| `refresh-rate` | Hz / 0 | 0 = auto-detección por XRandR |
| `detect-transient` | bool | Agrupa ventanas hijas (`WM_TRANSIENT_FOR`) con su padre: dialogs comparten foco/opacidad |
| `detect-client-leader` | bool | Ídem vía `WM_CLIENT_LEADER` (prioridad menor que transient) |
| `use-damage` | bool | **false**: repinta la pantalla completa en cada cambio en vez de la región dañada — más caro pero evita artefactos de blur (picom recomienda `resize-damage` como alternativa) |
| `log-level` | trace/debug/info/warn/error | `warn` — diagnóstico con `log-level = "debug"` + `log-file` |

> [!tip] `use-damage` y su coste
> `use-damage = true` (el defecto moderno) es el mayor ahorro de CPU de picom: solo repinta la ventana dañada. Se desactiva típicamente cuando aparecen **artefactos** (fantasmas al mover ventanas con blur). Si la máquina rinde mal, probar `use-damage = true` antes que bajar la calidad visual.

Otras opciones presentes en el archivo pero en su valor por defecto (comentadas): `daemon`, `dbus`, `sw-opti`, `use-ewmh-active-win`, `unredir-if-possible` (+delay/exclude), `resize-damage`, `invert-color-include`, `glx-no-stencil` (≤15 % más rápido, recomendado), `glx-no-rebind-pixmap` (rompe en drivers LLVMpipe/intel), `xrender-sync-fence` (necesario en algunos NVIDIA+GLX), `glx-fshader-win`, `force-win-blend`, `no-ewmh-fullscreen`, `max-brightness`, `transparent-clipping`, `show-all-xerrors`, `write-pid-path`.

### wintypes: política por tipo de ventana

```ini
wintypes:
{
  tooltip = { fade = true; shadow = true; shadow-radius = 0; shadow-opacity = 1.0; shadow-offset-x = -20; shadow-offset-y = -20; opacity = 0.8; full-shadow = true; };
  dnd = { shadow = false; }
  dropdown_menu = { shadow = false; };
  popup_menu    = { shadow = false; };
  utility       = { shadow = false; };
}
```

Bloque especial en sintaxis libconfig (`{}` con `;`). Cada **tipo EWMH** recibe su política: tooltips con fade y opacidad 0.8, menús popups/dropdowns y ventanas `utility` **sin sombra**, `dnd` sin sombra. Los 15 tipos disponibles: `unknown, desktop, dock, toolbar, menu, utility, splash, dialog, normal, dropdown_menu, popup_menu, tooltip, notification, combo, dnd`. Por tipo admite: `fade`, `shadow` (+`shadow-*`), `opacity`, `focus`, `full-shadow` (sombra bajo zonas transparentes), `redir-ignore` (con `unredir-if-possible`, evita re-redirección por este tipo).

> [!info] Sintaxis de todo el archivo
> picom usa **libconfig**: `clave = valor;` con punto y coma obligatorio, strings con `'` o `"`, enteros/floats, y listas `[ "a", "b" ]`. Un `;` olvidado produce `Error of config file "...": syntax error` al arrancar — la causa nº 1 de "picom no inicia".

### picom.conf completo (copia y pega)

El archivo personal íntegro — todos los valores anteriores, con los comentarios por defecto de picom como referencia:

```bash
##############################################################################
#                                  CORNERS                                   #
##############################################################################
# requires: <https://github.com/sdhand/compton>
corner-radius = 20;
rounded-corners-exclude = [
  #"window_type = 'normal'",
  #"class_g = 'firefox'",
];

round-borders = 20;
round-borders-exclude = [
  #"class_g = 'TelegramDesktop'",
];

# Specify a list of border width rules, in the format `PIXELS:PATTERN`, 
# Note we don't make any guarantee about possible conflicts with the
# border_width set by the window manager.
#
# example:
#    round-borders-rule = [ "2:class_g = 'URxvt'" ];
#
round-borders-rule = [];

##############################################################################
#                                  SHADOWS                                   #
##############################################################################

# Enabled client-side shadows on windows. Note desktop windows 
# (windows with '_NET_WM_WINDOW_TYPE_DESKTOP') never get shadow, 
# unless explicitly requested using the wintypes option.
#
shadow = true

# The blur radius for shadows, in pixels. (defaults to 12)
shadow-radius = 15

# The opacity of shadows. (0.0 - 1.0, defaults to 0.75)
shadow-opacity = .75

# The left offset for shadows, in pixels. (defaults to -15)
shadow-offset-x = -15

# The top offset for shadows, in pixels. (defaults to -15)
shadow-offset-y = -15

# Avoid drawing shadows on dock/panel windows. This option is deprecated,
# you should use the *wintypes* option in your config file instead.
#
# no-dock-shadow = false

# Don't draw shadows on drag-and-drop windows. This option is deprecated, 
# you should use the *wintypes* option in your config file instead.
#
# no-dnd-shadow = false

# Red color value of shadow (0.0 - 1.0, defaults to 0).
# shadow-red = .18

# Green color value of shadow (0.0 - 1.0, defaults to 0).
# shadow-green = .19

# Blue color value of shadow (0.0 - 1.0, defaults to 0).
# shadow-blue = .20

# Do not paint shadows on shaped windows. Note shaped windows 
# here means windows setting its shape through X Shape extension. 
# Those using ARGB background is beyond our control. 
# Deprecated, use 
#   shadow-exclude = 'bounding_shaped'
# or 
#   shadow-exclude = 'bounding_shaped && !rounded_corners'
# instead.
#
# shadow-ignore-shaped = ''

# Specify a list of conditions of windows that should have no shadow.
#
# examples:
#   shadow-exclude = "n:e:Notification";
#
# shadow-exclude = []
shadow-exclude = [
    "class_g = 'firefox' && argb"
];

# Specify a X geometry that describes the region in which shadow should not
# be painted in, such as a dock window region. Use 
#    shadow-exclude-reg = "x10+0+0"
# for example, if the 10 pixels on the bottom of the screen should not have shadows painted on.
#
# shadow-exclude-reg = "" 

# Crop shadow of a window fully on a particular Xinerama screen to the screen.
# xinerama-shadow-crop = false

##############################################################################
#                                  FADING                                    #
##############################################################################

# Fade windows in/out when opening/closing and when opacity changes,
#  unless no-fading-openclose is used.
#fading = true

# Opacity change between steps while fading in. (0.01 - 1.0, defaults to 0.028)
# fade-in-step = 0.028
fade-in-step = 0.01;

# Opacity change between steps while fading out. (0.01 - 1.0, defaults to 0.03)
# fade-out-step = 0.03
fade-out-step = 0.01;

# The time between steps in fade step, in milliseconds. (> 0, defaults to 10)
# fade-delta = 10

# Specify a list of conditions of windows that should not be faded.
# fade-exclude = []

# Do not fade on window open/close.
# no-fading-openclose = false

# Do not fade destroyed ARGB windows with WM frame. Workaround of bugs in Openbox, Fluxbox, etc.
# no-fading-destroyed-argb = false

##############################################################################
#                                   OPACITY                                  #
##############################################################################

# Opacity of inactive windows. (0.1 - 1.0, defaults to 1.0)
inactive-opacity = 1.0

# Opacity of window titlebars and borders. (0.1 - 1.0, disabled by default)
frame-opacity = 1.0

# Default opacity for dropdown menus and popup menus. (0.0 - 1.0, defaults to 1.0)
opacity = 1.0

# Let inactive opacity set by -i override the '_NET_WM_OPACITY' values of windows.
# inactive-opacity-override = true
inactive-opacity-override = false;

# Default opacity for active windows. (0.0 - 1.0, defaults to 1.0)
active-opacity = 1.0

# Dim inactive windows. (0.0 - 1.0, defaults to 0.0)
# inactive-dim = 0.0

# Specify a list of conditions of windows that should always be considered focused.
# focus-exclude = []
focus-exclude = [ "class_g = 'Cairo-clock'" ];

# Use fixed inactive dim value, instead of adjusting according to window opacity.
# inactive-dim-fixed = 1.0

# Specify a list of opacity rules, in the format `PERCENT:PATTERN`, 
# like `50:name *= "Firefox"`. picom-trans is recommended over this. 
# Note we don't make any guarantee about possible conflicts with other 
# programs that set '_NET_WM_WINDOW_OPACITY' on frame or client windows.
# example:
#    opacity-rule = [ "80:class_g = 'URxvt'" ];
#
# opacity-rule = []

# opacity-rule = [ "98:class_g = 'Polybar'" ]

##############################################################################
#                                    BLUR                                    #
##############################################################################

# Parameters for background blurring, see the *BLUR* section for more information.
blur-method = "dual_kawase"
blur-size = 2
blur-strength = 3

# Blur background of semi-transparent / ARGB windows. 
# Bad in performance, with driver-dependent behavior. 
# The name of the switch may change without prior notifications.
#
blur-background = true

# Blur background of windows when the window frame is not opaque. 
# Implies:
#    blur-background 
# Bad in performance, with driver-dependent behavior. The name may change.
#
#blur-background-frame = false

# Use fixed blur strength rather than adjusting according to window opacity.
#blur-background-fixed = false

# Specify the blur convolution kernel, with the following format:
# example:
#   blur-kern = "5,5,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1";
#
# blur-kern = ''
# blur-kern = "3x3box";

# Exclude conditions for background blur.
# blur-background-exclude = []
#blur-background-exclude = [
#    "! name~=''",
#    "name *= 'slop'",
#    "window_type = 'dock'",
#    "window_type = 'desktop'",
#    "_GTK_FRAME_EXTENTS@:c"
#];

##############################################################################
#                                    GENERAL                                 #
##############################################################################

# Daemonize process. Fork to background after initialization. Causes issues with certain (badly-written) drivers.
# daemon = false

# Specify the backend to use: `xrender`, `glx`, or `xr_glx_hybrid`.
# `xrender` is the default one.
#
# backend = 'glx'
backend = "glx";

# Enable/disable VSync.
# vsync = false
vsync = false

# Enable remote control via D-Bus. See the *D-BUS API* section below for more details.
# dbus = false

# Try to detect WM windows (a non-override-redirect window with no 
# child that has 'WM_STATE') and mark them as active.
#
# mark-wmwin-focused = false
mark-wmwin-focused = true;

# Mark override-redirect windows that doesn't have a child window with 'WM_STATE' focused.
# mark-ovredir-focused = false
mark-ovredir-focused = true;

# Try to detect windows with rounded corners and don't consider them 
# shaped windows. The accuracy is not very high, unfortunately.
#
# detect-rounded-corners = false
detect-rounded-corners = true;

# Detect '_NET_WM_OPACITY' on client windows, useful for window managers
# not passing '_NET_WM_OPACITY' of client windows to frame windows.
#
# detect-client-opacity = false
detect-client-opacity = true;

# Specify refresh rate of the screen. If not specified or 0, picom will 
# try detecting this with X RandR extension.
#
# refresh-rate = 60
refresh-rate = 0

# Limit picom to repaint at most once every 1 / 'refresh_rate' second to 
# boost performance. This should not be used with 
#   vsync drm/opengl/opengl-oml
# as they essentially does sw-opti's job already, 
# unless you wish to specify a lower refresh rate than the actual value.
#
# sw-opti = 

# Use EWMH '_NET_ACTIVE_WINDOW' to determine currently focused window, 
# rather than listening to 'FocusIn'/'FocusOut' event. Might have more accuracy, 
# provided that the WM supports it.
#
# use-ewmh-active-win = false

# Unredirect all windows if a full-screen opaque window is detected, 
# to maximize performance for full-screen windows. Known to cause flickering 
# when redirecting/unredirecting windows.
#
# unredir-if-possible = false

# Delay before unredirecting the window, in milliseconds. Defaults to 0.
# unredir-if-possible-delay = 0

# Conditions of windows that shouldn't be considered full-screen for unredirecting screen.
# unredir-if-possible-exclude = []

# Use 'WM_TRANSIENT_FOR' to group windows, and consider windows 
# in the same group focused at the same time.
#
# detect-transient = false
detect-transient = true

# Use 'WM_CLIENT_LEADER' to group windows, and consider windows in the same 
# group focused at the same time. 'WM_TRANSIENT_FOR' has higher priority if 
# detect-transient is enabled, too.
#
# detect-client-leader = false
detect-client-leader = true

# Resize damaged region by a specific number of pixels. 
# A positive value enlarges it while a negative one shrinks it. 
# If the value is positive, those additional pixels will not be actually painted 
# to screen, only used in blur calculation, and such. (Due to technical limitations, 
# with use-damage, those pixels will still be incorrectly painted to screen.) 
# Primarily used to fix the line corruption issues of blur, 
# in which case you should use the blur radius value here 
# (e.g. with a 3x3 kernel, you should use `--resize-damage 1`, 
# with a 5x5 one you use `--resize-damage 2`, and so on). 
# May or may not work with *--glx-no-stencil*. Shrinking doesn't function correctly.
#
# resize-damage = 1

# Specify a list of conditions of windows that should be painted with inverted color. 
# Resource-hogging, and is not well tested.
#
# invert-color-include = []

# GLX backend: Avoid using stencil buffer, useful if you don't have a stencil buffer. 
# Might cause incorrect opacity when rendering transparent content (but never 
# practically happened) and may not work with blur-background. 
# My tests show a 15% performance boost. Recommended.
#
# glx-no-stencil = false

# GLX backend: Avoid rebinding pixmap on window damage. 
# Probably could improve performance on rapid window content changes, 
# but is known to break things on some drivers (LLVMpipe, xf86-video-intel, etc.).
# Recommended if it works.
#
# glx-no-rebind-pixmap = false

# Disable the use of damage information. 
# This cause the whole screen to be redrawn everytime, instead of the part of the screen
# has actually changed. Potentially degrades the performance, but might fix some artifacts.
# The opposing option is use-damage
#
# no-use-damage = false
use-damage = false

# Use X Sync fence to sync clients' draw calls, to make sure all draw 
# calls are finished before picom starts drawing. Needed on nvidia-drivers 
# with GLX backend for some users.
#
# xrender-sync-fence = false

# GLX backend: Use specified GLSL fragment shader for rendering window contents. 
# See `compton-default-fshader-win.glsl` and `compton-fake-transparency-fshader-win.glsl` 
# in the source tree for examples.
#
# glx-fshader-win = ''

# Force all windows to be painted with blending. Useful if you 
# have a glx-fshader-win that could turn opaque pixels transparent.
#
# force-win-blend = false

# Do not use EWMH to detect fullscreen windows. 
# Reverts to checking if a window is fullscreen based only on its size and coordinates.
#
# no-ewmh-fullscreen = false

# Dimming bright windows so their brightness doesn't exceed this set value. 
# Brightness of a window is estimated by averaging all pixels in the window, 
# so this could comes with a performance hit. 
# Setting this to 1.0 disables this behaviour. Requires --use-damage to be disabled. (default: 1.0)
#
# max-brightness = 1.0

# Make transparent windows clip other windows like non-transparent windows do,
# instead of blending on top of them.
#
# transparent-clipping = false

# Set the log level. Possible values are:
#  "trace", "debug", "info", "warn", "error"
# in increasing level of importance. Case doesn't matter. 
# If using the "TRACE" log level, it's better to log into a file 
# using *--log-file*, since it can generate a huge stream of logs.
#
# log-level = "debug"
log-level = "warn";

# Set the log file.
# If *--log-file* is never specified, logs will be written to stderr. 
# Otherwise, logs will to written to the given file, though some of the early 
# logs might still be written to the stderr. 
# When setting this option from the config file, it is recommended to use an absolute path.
#
# log-file = '/path/to/your/log/file'

# Show all X errors (for debugging)
# show-all-xerrors = false

# Write process ID to a file.
# write-pid-path = '/path/to/your/log/file'

# Window type settings
# 
# 'WINDOW_TYPE' is one of the 15 window types defined in EWMH standard: 
#     "unknown", "desktop", "dock", "toolbar", "menu", "utility", 
#     "splash", "dialog", "normal", "dropdown_menu", "popup_menu", 
#     "tooltip", "notification", "combo", and "dnd".
# 
# Following per window-type options are available: ::
# 
#   fade, shadow:::
#     Controls window-type-specific shadow and fade settings.
# 
#   opacity:::
#     Controls default opacity of the window type.
# 
#   focus:::
#     Controls whether the window of this type is to be always considered focused. 
#     (By default, all window types except "normal" and "dialog" has this on.)
# 
#   full-shadow:::
#     Controls whether shadow is drawn under the parts of the window that you 
#     normally won't be able to see. Useful when the window has parts of it 
#     transparent, and you want shadows in those areas.
# 
#   redir-ignore:::
#     Controls whether this type of windows should cause screen to become 
#     redirected again after been unredirected. If you have unredir-if-possible
#     set, and doesn't want certain window to cause unnecessary screen redirection, 
#     you can set this to `true`.
#
wintypes:
{
  tooltip = { fade = true; shadow = true; shadow-radius = 0; shadow-opacity = 1.0; shadow-offset-x = -20; shadow-offset-y = -20; opacity = 0.8; full-shadow = true; }; 
  dnd = { shadow = false; }
  dropdown_menu = { shadow = false; };
  popup_menu    = { shadow = false; };
  utility       = { shadow = false; };
}
```

## Integración con bspwm

```bash
echo 'picom &' >> ~/.config/bspwm/bspwmrc
```

> [!warning] El `&` al final
> Indica ejecución en segundo plano. Sin él, `bspwmrc` esperaría a que picom terminara (nunca ocurre: es un proceso persistente) y el WM quedaría bloqueado a medio arrancar.
>
> [!tip] Reiniciar
> Tras cambios: `bspc wm -r` (o `super + shift + r` con la config personal de [[Configuración de BSPWM y SXHKD]]) reinicia bspwm y relanza picom. Para probar sin reiniciar: `pkill picom; picom --config ~/.config/picom/picom.conf -b` (`-b` = daemonize).

## Troubleshooting

| Síntoma | Causa probable | Fix |
|---|---|---|
| Picom no arranca; log `syntax error` | `;` o lista mal formada | `picom --config ~/.config/picom/picom.conf -b` en foreground para ver el error exacto |
| Tearing | `vsync = false` | `vsync = true` |
| FPS bajos en vídeo/juegos | Composición activa siempre | `unredir-if-possible = true` (aceptar parpadeos ocasionales) |
| Fantasmas/artefactos al mover ventanas | `use-damage` + blur | `use-damage = false` (ya está así en la config) o `resize-damage` |
| Negro/flicker en NVIDIA | Falta sync fence | `xrender-sync-fence = true` |
| Ventanas " pegajosas"/sin fade | Falta `mark-*-focused` | Activar las cuatro detecciones de la sección General |
| Consumo alto con blur | Fuerza excesiva | Bajar `blur-strength` (la config usa 3, casi decorativo) |

Logs: `picom --log-level debug` en foreground, o `log-file = '/ruta/log'` (ruta absoluta recomendada).

## Picom vs compositores Wayland

Con **Wayland el compositor es obligatorio y fusionado con el display server**: no existe un "picom para Wayland" porque el compositor *es* el servidor gráfico. Cada WM de Wayland trae sus efectos: **Sway** (familia wlroots, sucesor de i3) permite blur/opacidad por config pero no sombras animadas; **KWin**, **Mutter** y **Hyprland** incluyen blur, rounded corners y animaciones integrados. La pila de estas notas (bspwm + sxhkd + picom + polybar) es **X11 pura** — migrar a Wayland significaría reemplazar bspwm→sway/hyprland, polybar→waybar, sxhkd→fuzzel/shortcuts del compositor, y picom→config del propio compositor.

## Cheatsheet

| Sección | Opciones clave | Defecto |
|---|---|---|
| Esquinas | `corner-radius`, `rounded-corners-exclude`, `round-borders` | 0 (off) |
| Sombras | `shadow`, `shadow-radius/opacity/offset-x/offset-y`, `shadow-exclude` | off/12/0.75/−15/−15 |
| Fading | `fading`, `fade-in/out-step`, `fade-delta`, `fade-exclude` | off/0.028/0.03/10 ms |
| Opacidad | `inactive-opacity`, `active-opacity`, `frame-opacity`, `opacity-rule` | 1.0 |
| Blur | `blur-method`, `blur-size`, `blur-strength`, `blur-kern`, `blur-background` | none |
| Render | `backend`, `vsync`, `use-damage`, `unredir-if-possible` | xrender/false/true/false |
| Detección | `mark-wmwin-focused`, `detect-transient`, `detect-client-opacity` | false… |
| Diagnóstico | `log-level`, `log-file`, `show-all-xerrors` | info |
| Política | `wintypes { ... }` | por defecto EWMH |

## Recursos

- Repositorio oficial: <https://github.com/yshui/picom> — wiki con blur, animations (rama experimental `next`) y guías de rendimiento.
- Opciones completas: <https://wiki.archlinux.org/title/Picom> (tabla exhaustiva actualizada por versión).
- dual_kawase original: fork de ibhagwan (fusionado upstream).

## Relacionadas

- [[Configuración de BSPWM y SXHKD]] — desde dónde se lanza picom (`bspwmrc`)
- [[Instalación de Polybar y Picom]] — compilación de picom desde código fuente
- [[Configuración de Polybar]] — la otra pieza visual de la sesión
- [[Instalación y Configuración de Blue Sky Polybar]] — tema que aprovecha la transparencia
- [[Configuración Post-Instalación de Entorno de Escritorio]] — checklist donde picom se integra
