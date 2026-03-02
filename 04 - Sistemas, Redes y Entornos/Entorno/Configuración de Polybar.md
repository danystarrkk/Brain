---
aliases:
  - Polybar Config
  - Configuración Polybar
tags:
  - linux/customizacion
  - scripting/bash
  - hacking/laboratorios
estado: 🟢 Terminado
---
# Configuración de Polybar

Continuando desde, a continuación se presenta el contenido de los archivos de configuración de Polybar, listos para copiar y pegar. Estos archivos permiten personalizar la barra de estado del sistema para entornos de ciberseguridad ofensiva, incluyendo la visualización de información de red y objetivos de hacking.

> [!info] ¿Qué es Polybar?
> Polybar es una barra de estado rápida y altamente personalizable para Linux, diseñada para mostrar información del sistema y del entorno de trabajo. Es comúnmente utilizada con gestores de ventanas tipo *tiling* como BSPWM o i3, permitiendo a los usuarios crear barras de estado estéticas y funcionales con módulos personalizados.

## `current.ini`

Este archivo contiene la configuración principal de Polybar, definiendo la apariencia general de las barras, las fuentes, los colores y los módulos que se mostrarán.

```ini
;; _-_-_-_-_-_-_-_-_-_-_-_-_-_-_-_-_-_-_-_-_-_
;;
;;    ____        __      __              
;;   / __ \____  / /_  __/ /_  ____ ______
;;  / /_/ / __ \/ / / / / __ \/ __ `/ ___/
;;  / ____/ /_/ / / /_/ / /_/ / /_/ / /    
;;  /_/    \____/_/\__, /_.___/\__,_/_/     
;;              /____/                    
;;
;; _-_-_-_-_-_-_-_-_-_-_-_-_-_-_-_-_-_-_-_-_-_

;; Global WM Settings

[global/wm]
; Ajusta el valor superior de _NET_WM_STRUT_PARTIAL
; Usado para barras alineadas en la parte superior
margin-bottom = 0

; Ajusta el valor inferior de _NET_WM_STRUT_PARTIAL
; Usado para barras alineadas en la parte inferior
margin-top = 0

;; _-_-_-_-_-_-_-_-_-_-_-_-_-_-_-_-_-_-_-_-_-_

;; File Inclusion
; Incluye un archivo externo, como un archivo de módulo, etc.

include-file = ~/.config/polybar/colors.ini

;; _-_-_-_-_-_-_-_-_-_-_-_-_-_-_-_-_-_-_-_-_-_

;; Bar Settings

[bar/main]
; Utilice cualquiera de los siguientes comandos para listar las salidas disponibles:
; Si no se especifica, la aplicación elegirá la primera que encuentre.
; $ polybar -m | cut -d ':' -f 1
; $ xrandr -q | grep " connected" | cut -d ' ' -f1
monitor =

; Utilice el monitor especificado como respaldo si el principal no se encuentra.
monitor-fallback =

; Requiere que el monitor esté en estado conectado
; XRandR a veces reporta mi monitor como desconectado (cuando está en uso)
monitor-strict = false

; Indica al gestor de ventanas que no configure la ventana.
; Utilice esto para desvincular la barra si su WM está bloqueando su tamaño/posición.
;override-redirect = true

; Coloca la barra en la parte inferior de la pantalla
bottom = true

; Prefiere una posición central fija para el bloque `modules-center`
; Cuando es falso, la posición central se basará en el tamaño de los otros bloques.
fixed-center = true

; Dimensiones definidas como valor en píxeles (ej. 35) o porcentaje (ej. 50%).
; El porcentaje puede extenderse opcionalmente con un desplazamiento en píxeles, por ejemplo:
; 50%:-10, lo que resultará en un ancho o alto del 50% menos 10 píxeles
width = 17%
height = 60

; Desplazamiento definido como valor en píxeles (ej. 35) o porcentaje (ej. 50%)
; El porcentaje puede extenderse opcionalmente con un desplazamiento en píxeles, por ejemplo:
; 50%:-10, lo que resultará en un desplazamiento en la dirección X o Y
; del 50% menos 10 píxeles
offset-x = 20
offset-y = 20

; Color de fondo ARGB (ej. #f00, #ff992a, #ddff1023)
background = ${color.bg}

; Color de primer plano ARGB (ej. #f00, #ff992a, #ddff1023)
foreground = ${color.fg}

; Gradiente de fondo (pasos verticales)
;   background-[0-9]+ = #aarrggbb
;;background-0 = 

; Valor utilizado para dibujar esquinas redondeadas.
; Nota: Esto no debe usarse junto con `border-size` ya que el borde
; no se redondeará.
; Los valores individuales superior/inferior se pueden definir usando:
;   radius-{top,bottom}
radius-top = 10.0
radius-bottom = 10.0

; Tamaño en píxeles y color ARGB de la línea inferior/superior.
; Los valores individuales se pueden definir usando:
;   {overline,underline}-size
;   {overline,underline}-color
line-size = 2
line-color = ${color.ac}

; Valores aplicados a todos los bordes.
; Los valores individuales de cada lado se pueden definir usando:
;   border-{left,top,right,bottom}-size
;   border-{left,top,right,bottom}-color
; Los bordes superior e inferior se añaden a la altura de la barra, por lo que la altura efectiva
; de la ventana es:
;   height + border-top-size + border-bottom-size
; Mientras tanto, el ancho efectivo de la ventana se define completamente por la clave `width` y
; el borde se coloca dentro de esta área. Por lo tanto, el espacio horizontal efectivo
; en la barra es:
;   width - border-right-size - border-left-size
border-bottom-size = 0
border-color = ${color.ac}

; Número de espacios a añadir al principio/final de la barra.
; Los valores individuales de cada lado se pueden definir usando:
;   padding-{left,right}
padding = 2

; Número de espacios a añadir antes/después de cada módulo.
; Los valores individuales de cada lado se pueden definir usando:
;   module-margin-{left,right}
module-margin-left = 1
module-margin-right = 1

; Las fuentes se definen usando <nombre-de-fuente>;<desplazamiento-vertical>.
; Los nombres de las fuentes se especifican usando un patrón de Fontconfig.
;   font-0 = NotoSans-Regular:size=8;2
;   font-1 = MaterialIcons:size=10
;   font-2 = Termsynu:size=8;-1
;   font-3 = FontAwesome:size=10
; Consulte la página wiki de Fuentes para más detalles.

font-0 = "Iosevka Nerd Font:size=13;3"
font-1 = "Iosevka Nerd Font:bold:size=24;2"
font-2 = "Iosevka Nerd Font:size=22;6"
font-3 = "Source Code Pro:size=10;3"
font-5 = "Helvetica Rounded:style=Bold:size=10.5;3"
font-4 = "Montserrat Medium:style=Medium:size=12;3"
font-6 = "Hurmit Nerd Font Mono:style=medium:pixelsize=17;3"
font-7 = "Hack Nerd Font Mono:size=25;7"
; Los módulos se añaden a uno de los bloques disponibles:
;   modules-left = cpu ram
;   modules-center = xwindow xbacklight
;   modules-right = ipc clock

;; Módulos disponibles
;;
;alsa backlight battery
;bspwm cpu date
;filesystem github i3
;subscriber demo memory
;menu-apps mpd wired-network
;wireless-network network pulseaudio
;name_you_want temperature my-text-label
;backlight keyboard title workspaces 
;;
;; Módulos de usuario
;checknetwork updates window_switch launcher powermenu sysmenu menu
;;
;; Barras
;cpu_bar memory_bar filesystem_bar mpd_bar 
;volume brightness battery_bar 

;; _-_-_-_-_-_-_-_-_-_-_-_-_-_-_-_-_-_-_-_-_-_
[bar/primary]
inherit = bar/main
offset-x = 96.9%
offset-y = 15
width = 2.5%
height = 40
bottom = false
padding = 0
module-margin-left = 0
module-margin-right = 0
background = ${color.bg}
;foreground = ${color.blue}
foreground = #44abff
;modules-center = web sep2 files sep2 edit sep2 apps
modules-center = sysmenu
wm-restack = bspwm

[bar/ethernet_bar]
inherit = bar/main
width = 10%
height = 40
offset-x = 14.5%
offset-y = 15
background = ${color.bg}
foreground = ${color.white}
bottom = false
padding = 1
;padding-top = 2
module-margin-left = 0
module-margin-right = 0
;modules-left = date sep mpd
modules-center = ethernet_status
wm-restack = bspwm

[bar/target_hack]
inherit = bar/main
width = 17.6%
height = 40
offset-x = 81.54%
offset-y = 15
background = ${color.bg}
foreground = ${color.white}
bottom = false
padding = 1
;padding-top = 2
module-margin-left = 0
module-margin-right = 0
;modules-left = date sep mpd
modules-center = target_status
wm-restack = bspwm

[bar/vpn_bar]
inherit = bar/main
width = 10%
height = 40
offset-x = 71%
offset-y = 15
background = ${color.bg}
foreground = ${color.white}
bottom = false
padding = 1
;padding-top = 2
module-margin-left = 0
module-margin-right = 0
;modules-left = date sep mpd
modules-center = vpn_status
wm-restack = bspwm

[bar/secondary]
inherit = bar/main
width = 7%
height = 40
offset-x = 3.8%
offset-y = 15
background = ${color.bg}
foreground = ${color.white}
bottom = false
padding = 1
;padding-top = 2
module-margin-left = 0
module-margin-right = 0
;modules-left = date sep mpd
modules-center = date 
wm-restack = bspwm

[bar/log]
inherit = bar/main
width = 2.5%
height = 40
offset-x = 1%
offset-y = 15
bottom = false
foreground = ${color.white}
background = ${color.bg}
padding = 0
modules-center = my-text-label
wm-restack = bspwm
;override-redirect = true

[bar/top]
inherit = bar/main
width = 6%
height = 40
offset-x = 90.5%
offset-y = 15
bottom = false
font-0 = "Iosevka Nerd Font:size=12;3"
background = ${color.bg}
padding = 1
module-margin-left = 1
module-margin-right = 1
;wm-restack = bspwm
modules-center = alsa battery network

;; _-_-_-_-_-_-_-_-_-_-_-_-_-_-_-_-_-_-_-_-_-_

; El separador se insertará entre la salida de cada módulo
separator =

; Este valor se utiliza para añadir espacio extra entre elementos
; @deprecated: Este parámetro será eliminado en una próxima versión
spacing = 0

; Valor de opacidad entre 0.0 y 1.0 utilizado para el efecto de aparición/desaparición
dim-value = 1.0

; Valor a utilizar para establecer el átomo WM_NAME
; Si el valor está vacío o indefinido, el valor del átomo
; se creará a partir de la siguiente plantilla: polybar-[BAR]_[MONITOR]
; NOTA: Los marcadores de posición no están disponibles para valores personalizados
wm-name = openbox

; Configuración regional utilizada para localizar varios datos de módulos (ej. fecha)
; Espera una configuración regional de libc válida, por ejemplo: sv_SE.UTF-8
locale = 

; Posición de la ventana de la bandeja del sistema
; Si está vacía o indefinida, el soporte de la bandeja se deshabilitará
; NOTA: Una bandeja alineada al centro cubrirá los módulos alineados al centro
;
; Posiciones disponibles:
;   left
;   center
;   right
;   none
tray-position = none

; Si es true, la barra no desplazará su
; contenido cuando la bandeja cambie
tray-detached = false

; Tamaño máximo del icono de la bandeja
tray-maxsize = 16

; ¡DEPRECADO! Desde la versión 3.3.0, la bandeja siempre utiliza pseudo-transparencia
; Habilita la pseudo-transparencia
; Se habilitará automáticamente si se define un color de fondo
; completamente transparente usando `tray-background`
tray-transparent = false

; Color de fondo para el contenedor de la bandeja
; Color ARGB (ej. #f00, #ff992a, #ddff1023)
; Por defecto, el contenedor de la bandeja utilizará el
; color de fondo de la barra.
tray-background = ${color.bg}

; Desplazamiento de la bandeja definido como valor en píxeles (ej. 35) o porcentaje (ej. 50%)
tray-offset-x = 0
tray-offset-y = 0

; Relleno a los lados de cada icono de la bandeja
tray-padding = 0

; Factor de escala para los clientes de la bandeja
tray-scale = 1.0

; Reapila la ventana de la barra y la coloca por encima de la
; raíz del gestor de ventanas seleccionado
;
; Soluciona el problema donde la barra se dibuja
; encima de las ventanas en pantalla completa
;
; Gestores de ventanas actualmente soportados:
;   bspwm
;   i3 (requiere: `override-redirect = true`)
wm-restack = bspwm

; Establece valores de DPI utilizados al renderizar texto
; Esto solo afecta a las fuentes escalables
; dpi = 

; Habilita el soporte para la mensajería entre procesos
; Consulte la página wiki de Mensajería para más detalles.
enable-ipc = true

; Manejadores de clic de respaldo que se llamarán si
; no se encuentra un manejador de módulo coincidente.
click-left = 
click-middle = 
click-right =
scroll-up =
scroll-down =
double-click-left =
double-click-middle =
double-click-right =

; Requiere que Polybar se compile con soporte xcursor (xcb-util-cursor)
; Los valores posibles son:
; - default   : El puntero predeterminado como antes, también puede ser una cadena vacía (predeterminado)
; - pointer   : Típicamente en forma de mano
; - ns-resize : Flechas arriba y abajo, se pueden usar para indicar desplazamiento
cursor-click = 
cursor-scroll = 

;; WM Workspace Specific

; bspwm
;;scroll-up = bspwm-desknext
;;scroll-down = bspwm-deskprev
;;scroll-up = bspc desktop -f prev.local
;;scroll-down = bspc desktop -f next.local

;i3
;;scroll-up = i3wm-wsnext
;;scroll-down = i3wm-wsprev
;;scroll-up = i3-msg workspace next_on_output
;;scroll-down = i3-msg workspace prev_on_output

;openbox
;awesome
;etc

;; _-_-_-_-_-_-_-_-_-_-_-_-_-_-_-_-_-_-_-_-_-_

;; Application Settings

[settings]
; La configuración de throttle permite que el bucle de eventos 'trague' hasta X eventos
; si ocurren dentro de Y milisegundos después de que se recibió el primer evento.
; Esto se hace para prevenir una inundación de eventos de actualización.
;
; Por ejemplo, si 5 módulos emiten un evento de actualización al mismo tiempo, realmente
; solo nos importa el último. Pero si esperamos demasiado tiempo para que los eventos se 'traguen',
; la barra parecería lenta, por lo que continuamos si el tiempo de espera
; expira o se alcanza el límite.
throttle-output = 5
throttle-output-for = 10

; Tiempo en milisegundos que el manejador de entrada esperará entre el procesamiento de eventos
throttle-input-for = 30

; Recargar al recibir eventos XCB_RANDR_SCREEN_CHANGE_NOTIFY
screenchange-reload = false

; Operadores de composición
; @ver: <https://www.cairographics.org/manual/cairo-cairo-t.html#cairo-operator-t>
compositing-background = source
compositing-foreground = over
compositing-overline = over
compositing-underline = over
compositing-border = over

; Define valores de respaldo utilizados por todos los formatos de módulo
format-foreground = 
format-background =
format-underline =
format-overline =
format-spacing =
format-padding =
format-margin =
format-offset =

; Habilita la pseudo-transparencia para la barra
; Si se establece en true, la barra puede ser transparente sin un compositor.
pseudo-transparency = false

;; _-_-_-_-_-_-_-_-_-_-_-_-_-_-_-_-_-_-_-_-_-_
;;
;;    __  ___          __      __         
;;   /  |/  /___  ____/ /_  __/ /__  _____
;;  / /|_/ / __ \/ __  / / / / / _ \/ ___/
;;  / /  / / /_/ / /_/ / /_/ / /  __(__  ) 
;;  /_/  /_/\____/\__,_/\__,_/_/\___/____/  
;;
;; _-_-_-_-_-_-_-_-_-_-_-_-_-_-_-_-_-_-_-_-_-_
[module/my-text-label]
type = custom/text
content = %{T7}󰌽
;interval = 1.0
;time = %k : %M
;date = %b %e
;format = <label1>
;format-foreground = ${color.white}
;label1 = aadasd
;label1-font = 7

;; _-_-_-_-_-_-_-_-_-_-_-_-_-_-_-_-_-_-_-_-_-_

[module/date]
type = internal/date

interval = 1.0
time = %k : %M
date = %b %e
;padding-top = 2
format = <label>
format-foreground = ${color.white}
;format-background = $(color.blue}
;label = %{T7} %{T6}%date%|%time%
;label = %{T7} %{T6}Pop! OS  |   %time%
label = %date%  |  %time% 
label-font = 6

;; _-_-_-_-_-_-_-_-_-_-_-_-_-_-_-_-_-_-_-_-_-_

[module/mpd]
type = internal/mpd

interval = 2

format-online = <label-song>
format-online-foreground = ${color.fg}

label-song = "%title%"
label-song-maxlen = 12
label-song-ellipsis = true
label-offline = "Offline"

;; _-_-_-_-_-_-_-_-_-_-_-_-_-_-_-_-_-_-_-_-_-_

;; Apps
 
[module/term]
type = custom/text

content = "%{T3}%{T-}"
content-foreground = ${color.black}
content-background = ${color.bg}
content-padding = 0

click-left  = termite &

[module/web]
type = custom/text

content = "%{T3}%{T-}"
content-foreground = ${color.white}
content-background = ${color.bg}
content-padding = 0

click-left  = firefox &

[module/files]
type = custom/text

content = "%{T3}%{T-}"
content-foreground = ${color.blue}
content-background = ${color.bg}
content-padding = 0

click-left  = thunar &

[module/edit]
type = custom/text

content = "%{T3}%{T-}"
content-foreground = ${color.blue-gray}
content-background = ${color.bg}
content-padding = 0

click-left  = geany &

[module/apps]
type = custom/text

content = "%{T3}%{T-}"
content-foreground = ${color.fg}
content-background = ${color.bg}
content-padding = 0

click-left  = ~/.config/polybar/scripts/launcher &

;; _-_-_-_-_-_-_-_-_-_-_-_-_-_-_-_-_-_-_-_-_-_

[module/sep]
type = custom/text
content = " | "
content-font = 1

;;content-background = #000
content-foreground = ${color.fg}
;;content-padding = 4

[module/sep2]
type = custom/text
content = "   "

;;content-background = #000
content-foreground = ${color.fg}
;;content-padding = 4

;; _-_-_-_-_-_-_-_-_-_-_-_-_-_-_-_-_-_-_-_-_-_

[module/alsa]
type = internal/alsa
;format-background = ${color.blue}
format-volume = <ramp-volume>
format-muted = <label-muted>
label-muted = %{A3:pavucontrol &:} 婢 %{A}
label-muted-foreground = ${color.gray}
;click-right = pavucontrol
ramp-volume-0 = %{A3:pavucontrol &:} 奄%{A}
ramp-volume-1 = %{A3:pavucontrol &:}奔%{A}
ramp-volume-2 = %{A3:pavucontrol &:}奔%{A}
ramp-volume-3 = %{A3:pavucontrol &:}墳%{A}
ramp-volume-4 = %{A3:pavucontrol &:}墳%{A}
ramp-volume-foreground = ${color.white}

;; _-_-_-_-_-_-_-_-_-_-_-_-_-_-_-_-_-_-_-_-_-_

[module/battery]
type = internal/battery

full-at = 99
battery = BAT1
adapter = ACAD

poll-interval = 2
time-format = %H:%M

format-charging = <animation-charging>
format-discharging = <ramp-capacity>

format-full = <label-full>
format-full-foreground = ${color.white}
;format-full-background = $(color.white}
label-full = 

ramp-capacity-0 = 
ramp-capacity-1 = 
ramp-capacity-2 = 
ramp-capacity-3 = 
ramp-capacity-4 = 
ramp-capacity-5 = 
ramp-capacity-6 = 
ramp-capacity-7 = 
ramp-capacity-8 = 
ramp-capacity-9 = 
ramp-capacity-foreground = ${color.white}

animation-charging-0 = 
animation-charging-1 = 
animation-charging-2 = 
animation-charging-3 = 
animation-charging-4 = 
animation-charging-5 = 
animation-charging-6 = 
animation-charging-7 = 
animation-charging-8 = 
animation-charging-9 = 
animation-charging-foreground = ${color.white}
animation-charging-framerate = 750

;; _-_-_-_-_-_-_-_-_-_-_-_-_-_-_-_-_-_-_-_-_-_

[module/network]
type = internal/network
interface = wlp2s0
click-left = kcmshell5 kcm_networkmanagement
interval = 1.0
accumulate-stats = true
unknown-as-up = true

format-connected = <label-connected>
format-connected-foreground = ${color.white}
;content-background = $(color.blue}
format-disconnected = <label-disconnected>
format-disconnected-foreground = ${color.gray}

label-connected = 直
label-disconnected = 睊

;; _-_-_-_-_-_-_-_-_-_-_-_-_-_-_-_-_-_-_-_-_-_

[module/sysmenu]
type = custom/text
content = 襤

;content-foreground = ${color.white}
click-left = ~/.config/polybar/scripts/powermenu_alt

[module/ethernet_status]
type = custom/script
interval = 2
exec = /home/stark/.config/bspwm/scripts/ethernet_status.sh

[module/vpn_status]
type = custom/script
interval = 2
exec = /home/stark/.config/bspwm/scripts/vpn_status.sh

[module/target_status]
type = custom/script
interval = 2
exec = /home/stark/.config/bspwm/scripts/target_hack.sh

;; _-_-_-_-_-_-_-_-_-_-_-_-_-_-_-_-_-_-_-_-_-_
;;    __________  ______
;;   / ____/ __ \/ ____/
;;  / __/ / / / / /_    
;;  / /___/ /_/ / __/    
;;  /_____/\____/_/       
;;
;; _-_-_-_-_-_-_-_-_-_-_-_-_-_-_-_-_-_-_-_-_-_
```

## `launch.sh`

Este script es responsable de iniciar las instancias de Polybar. Primero, termina cualquier instancia de Polybar que ya esté en ejecución para evitar duplicados, espera a que los procesos se cierren y luego lanza las barras configuradas.

```bash
#!/usr/bin/env sh

## Añadir esto a su archivo de inicio del gestor de ventanas.

# Terminar instancias de barra ya en ejecución
killall -q polybar

## Esperar hasta que los procesos se hayan cerrado
while pgrep -u $UID -x polybar >/dev/null; do sleep 1; done

## Lanzar

## Barra izquierda
polybar log -c ~/.config/polybar/current.ini &
polybar target_hack -c ~/.config/polybar/current.ini &
#polybar secondary -c ~/.config/polybar/current.ini &

## Barra derecha
#polybar top -c ~/.config/polybar/current.ini &
#polybar primary -c ~/.config/polybar/current.ini &
polybar ethernet_bar -c ~/.config/polybar/current.ini &
polybar vpn_bar -c ~/.config/polybar/current.ini &

## Barra central
polybar primary -c ~/.config/polybar/workspace.ini &
```

## `workspace.ini`

Este archivo de configuración está diseñado específicamente para la barra de espacios de trabajo (workspaces), permitiendo una visualización clara de los escritorios virtuales y su estado.

```ini
;; _-_-_-_-_-_-_-_-_-_-_-_-_-_-_-_-_-_-_-_-_-_
;;
;;    ____        __      __              
;;   / __ \____  / /_  __/ /_  ____ ______
;;  / /_/ / __ \/ / / / / __ \/ __ `/ ___/
;;  / ____/ /_/ / / /_/ / /_/ / /_/ / /    
;;  /_/    \____/_/\__, /_.___/\__,_/_/     
;;              /____/                    
;;
;; _-_-_-_-_-_-_-_-_-_-_-_-_-_-_-_-_-_-_-_-_-_

;; Global WM Settings

[global/wm]
; Ajusta el valor superior de _NET_WM_STRUT_PARTIAL
; Usado para barras alineadas en la parte superior
margin-bottom = 0

; Ajusta el valor inferior de _NET_WM_STRUT_PARTIAL
; Usado para barras alineadas en la parte inferior
margin-top = 0

;; _-_-_-_-_-_-_-_-_-_-_-_-_-_-_-_-_-_-_-_-_-_

;; File Inclusion
; Incluye un archivo externo, como un archivo de módulo, etc.

include-file = ~/.config/polybar/colors.ini

;; _-_-_-_-_-_-_-_-_-_-_-_-_-_-_-_-_-_-_-_-_-_

;; Bar Settings

[bar/main]
; Utilice cualquiera de los siguientes comandos para listar las salidas disponibles:
; Si no se especifica, la aplicación elegirá la primera que encuentre.
; $ polybar -m | cut -d ':' -f 1
; $ xrandr -q | grep " connected" | cut -d ' ' -f1
monitor =

; Utilice el monitor especificado como respaldo si el principal no se encuentra.
monitor-fallback =

; Requiere que el monitor esté en estado conectado
; XRandR a veces reporta mi monitor como desconectado (cuando está en uso)
monitor-strict = false

; Indica al gestor de ventanas que no configure la ventana.
; Utilice esto para desvincular la barra si su WM está bloqueando su tamaño/posición.
;override-redirect = true

; Coloca la barra en la parte inferior de la pantalla
;bottom = true

; Prefiere una posición central fija para el bloque `modules-center`
; Cuando es falso, la posición central se basará en el tamaño de los otros bloques.
fixed-center = true

; Dimensiones definidas como valor en píxeles (ej. 35) o porcentaje (ej. 50%).
; El porcentaje puede extenderse opcionalmente con un desplazamiento en píxeles, por ejemplo:
; 50%:-10, lo que resultará en un ancho o alto del 50% menos 10 píxeles
width = 10%
height = 40

; Desplazamiento definido como valor en píxeles (ej. 35) o porcentaje (ej. 50%)
; El porcentaje puede extenderse opcionalmente con un desplazamiento en píxeles, por ejemplo:
; 50%:-10, lo que resultará en un desplazamiento en la dirección X o Y
; del 50% menos 10 píxeles
offset-x = 20
offset-y = 20

; Color de fondo ARGB (ej. #f00, #ff992a, #ddff1023)
background = ${color.bg}
;background = #A7D6FD
; Color de primer plano ARGB (ej. #f00, #ff992a, #ddff1023)
foreground = ${color.fg}

; Gradiente de fondo (pasos verticales)
;   background-[0-9]+ = #aarrggbb
;;background-0 = 

; Valor utilizado para dibujar esquinas redondeadas.
; Nota: Esto no debe usarse junto con `border-size` ya que el borde
; no se redondeará.
; Los valores individuales superior/inferior se pueden definir usando:
;   radius-{top,bottom}
radius-top = 10.0
radius-bottom = 10.0

; Tamaño en píxeles y color ARGB de la línea inferior/superior.
; Los valores individuales se pueden definir usando:
;   {overline,underline}-size
;   {overline,underline}-color
line-size = 2
line-color = ${color.ac}

; Valores aplicados a todos los bordes.
; Los valores individuales de cada lado se pueden definir usando:
;   border-{left,top,right,bottom}-size
;   border-{left,top,right,bottom}-color
; Los bordes superior e inferior se añaden a la altura de la barra, por lo que la altura efectiva
; de la ventana es:
;   height + border-top-size + border-bottom-size
; Mientras tanto, el ancho efectivo de la ventana se define completamente por la clave `width` y
; el borde se coloca dentro de esta área. Por lo tanto, el espacio horizontal efectivo
; en la barra es:
;   width - border-right-size - border-left-size
border-size = 0
border-color = ${color.bg}

; Número de espacios a añadir al principio/final de la barra.
; Los valores individuales de cada lado se pueden definir usando:
;   padding-{left,right}
padding = 2

; Número de espacios a añadir antes/después de cada módulo.
; Los valores individuales de cada lado se pueden definir usando:
;   module-margin-{left,right}
module-margin-left = 1
module-margin-right = 1

; Las fuentes se definen usando <nombre-de-fuente>;<desplazamiento-vertical>.
; Los nombres de las fuentes se especifican usando un patrón de Fontconfig.
;   font-0 = NotoSans-Regular:size=8;2
;   font-1 = MaterialIcons:size=10
;   font-2 = Termsynu:size=8;-1
;   font-3 = FontAwesome:size=10
; Consulte la página wiki de Fuentes para más detalles.

font-0 = "Iosevka Nerd Font:size=18;5"
font-1 = "Iosevka Nerd Font:size=12;2"
font-2 = "Iosevka Nerd Font:bold:size=24;2"
font-3 = "Hack Nerd Font Mono:size=18;3"
; Los módulos se añaden a uno de los bloques disponibles:
;   modules-left = cpu ram
;   modules-center = xwindow xbacklight
;   modules-right = ipc clock

;; Módulos disponibles
;;
;alsa backlight battery
;bspwm cpu date
;filesystem github i3
;subscriber demo memory
;menu-apps mpd wired-network
;wireless-network network pulseaudio
;name_you_want temperature my-text-label
;backlight keyboard title workspaces 
;;
;; Módulos de usuario
;checknetwork updates window_switch launcher powermenu sysmenu menu
;;
;; Barras
;cpu_bar memory_bar filesystem_bar mpd_bar 
;volume brightness battery_bar 

;; _-_-_-_-_-_-_-_-_-_-_-_-_-_-_-_-_-_-_-_-_-_

[bar/primary]
inherit = bar/main

override-redirect = true
wm-restack = bspwm
offset-x = 4%
offset-y = 15
bottom = false
padding = 1
module-margin-left = 0
module-margin-right = 0

modules-center = sep2 workspaces

[bar/secondary]
inherit = bar/main
offset-x = 30
offset-y = 70
background = ${color.trans}
foreground = ${color.white}

padding = 1
module-margin-left = 0
module-margin-right = 0

modules-left = name sep title

;; _-_-_-_-_-_-_-_-_-_-_-_-_-_-_-_-_-_-_-_-_-_

; El separador se insertará entre la salida de cada módulo
separator =

; Este valor se utiliza para añadir espacio extra entre elementos
; @deprecated: Este parámetro será eliminado en una próxima versión
spacing = 0

; Valor de opacidad entre 0.0 y 1.0 utilizado para el efecto de aparición/desaparición
dim-value = 1.0

; Valor a utilizar para establecer el átomo WM_NAME
; Si el valor está vacío o indefinido, el valor del átomo
; se creará a partir de la siguiente plantilla: polybar-[BAR]_[MONITOR]
; NOTA: Los marcadores de posición no están disponibles para valores personalizados
wm-name = openbox

; Configuración regional utilizada para localizar varios datos de módulos (ej. fecha)
; Espera una configuración regional de libc válida, por ejemplo: sv_SE.UTF-8
locale = 

; Posición de la ventana de la bandeja del sistema
; Si está vacía o indefinida, el soporte de la bandeja se deshabilitará
; NOTA: Una bandeja alineada al centro cubrirá los módulos alineados al centro
;
; Posiciones disponibles:
;   left
;   center
;   right
;   none
tray-position = none

; Si es true, la barra no desplazará su
; contenido cuando la bandeja cambie
tray-detached = false

; Tamaño máximo del icono de la bandeja
tray-maxsize = 16

; ¡DEPRECADO! Desde la versión 3.3.0, la bandeja siempre utiliza pseudo-transparencia
; Habilita la pseudo-transparencia
; Se habilitará automáticamente si se define un color de fondo
; completamente transparente usando `tray-background`
tray-transparent = false

; Color de fondo para el contenedor de la bandeja
; Color ARGB (ej. #f00, #ff992a, #ddff1023)
; Por defecto, el contenedor de la bandeja utilizará el
; color de fondo de la barra.
tray-background = ${color.bg}

; Desplazamiento de la bandeja definido como valor en píxeles (ej. 35) o porcentaje (ej. 50%)
tray-offset-x = 0
tray-offset-y = 0

; Relleno a los lados de cada icono de la bandeja
tray-padding = 0

; Factor de escala para los clientes de la bandeja
tray-scale = 1.0

; Reapila la ventana de la barra y la coloca por encima de la
; raíz del gestor de ventanas seleccionado
;
; Soluciona el problema donde la barra se dibuja
; encima de las ventanas en pantalla completa
;
; Gestores de ventanas actualmente soportados:
;   bspwm
;   i3 (requiere: `override-redirect = true`)
wm-restack = bspwm

; Establece valores de DPI utilizados al renderizar texto
; Esto solo afecta a las fuentes escalables
; dpi = 

; Habilita el soporte para la mensajería entre procesos
; Consulte la página wiki de Mensajería para más detalles.
enable-ipc = true

; Manejadores de clic de respaldo que se llamarán si
; no se encuentra un manejador de módulo coincidente.
click-left = 
click-middle = 
click-right =
scroll-up =
scroll-down =
double-click-left =
double-click-middle =
double-click-right =

; Requiere que Polybar se compile con soporte xcursor (xcb-util-cursor)
; Los valores posibles son:
; - default   : El puntero predeterminado como antes, también puede ser una cadena vacía (predeterminado)
; - pointer   : Típicamente en forma de mano
; - ns-resize : Flechas arriba y abajo, se pueden usar para indicar desplazamiento
cursor-click = 
cursor-scroll = 

;; WM Workspace Specific

; bspwm
;;scroll-up = bspwm-desknext
;;scroll-down = bspwm-deskprev
;;scroll-up = bspc desktop -f prev.local
;;scroll-down = bspc desktop -f next.local

;i3
;;scroll-up = i3wm-wsnext
;;scroll-down = i3wm-wsprev
;;scroll-up = i3-msg workspace next_on_output
;;scroll-down = i3-msg workspace prev_on_output

;openbox
;awesome
;etc

;; _-_-_-_-_-_-_-_-_-_-_-_-_-_-_-_-_-_-_-_-_-_

;; Application Settings

[settings]
; La configuración de throttle permite que el bucle de eventos 'trague' hasta X eventos
; si ocurren dentro de Y milisegundos después de que se recibió el primer evento.
; Esto se hace para prevenir una inundación de eventos de actualización.
;
; Por ejemplo, si 5 módulos emiten un evento de actualización al mismo tiempo, realmente
; solo nos importa el último. Pero si esperamos demasiado tiempo para que los eventos se 'traguen',
; la barra parecería lenta, por lo que continuamos si el tiempo de espera
; expira o se alcanza el límite.
throttle-output = 5
throttle-output-for = 10

; Tiempo en milisegundos que el manejador de entrada esperará entre el procesamiento de eventos
throttle-input-for = 30

; Recargar al recibir eventos XCB_RANDR_SCREEN_CHANGE_NOTIFY
screenchange-reload = false

; Operadores de composición
; @ver: <https://www.cairographics.org/manual/cairo-cairo-t.html#cairo-operator-t>
compositing-background = source
compositing-foreground = over
compositing-overline = over
compositing-underline = over
compositing-border = over

; Define valores de respaldo utilizados por todos los formatos de módulo
format-foreground = 
format-background = 
format-underline =
format-overline =
format-spacing =
format-padding =
format-margin =
format-offset =

; Habilita la pseudo-transparencia para la barra
; Si se establece en true, la barra puede ser transparente sin un compositor.
pseudo-transparency = false

;; _-_-_-_-_-_-_-_-_-_-_-_-_-_-_-_-_-_-_-_-_-_
;;
;;    __  ___          __      __         
;;   /  |/  /___  ____/ /_  __/ /__  _____
;;  / /|_/ / __ \/ __  / / / / / _ \/ ___/
;;  / /  / / /_/ / /_/ / /_/ / /  __(__  ) 
;;  /_/  /_/\____/\__,_/\__,_/_/\___/____/  
;;
;; _-_-_-_-_-_-_-_-_-_-_-_-_-_-_-_-_-_-_-_-_-_

[module/workspaces]
type = internal/xworkspaces

pin-workspaces = true
enable-click = true
enable-scroll = true
font-0 = Material Icons:style=Regular
font-1 = FontAwesome5Free:style=Solid:pixelsize=10:antialias=false;3
font-2 = FontAwesome5Brands:style=Solid:pixelsize=10:antialias=false;3
;icon-0 = 1;
icon-0 = 1;-
icon-1 = 2;
icon-2 = 3;
icon-3 = 4;
icon-4 = 5;
;icon-default = 
;icon-default = ─
icon-default = ∙
format = <label-state>
format-padding = 0

;label-active = ""
label-active = " "
label-active-foreground = #81a1c1
label-active-background = ${color.bg}

label-occupied = " "
label-occupied-foreground = #bf616a
label-occupied-background = ${color.bg}

label-urgent = "%icon% "
label-urgent-foreground = ${color.ac}
label-urgent-background = ${color.bg}

label-empty = " "
label-empty-foreground = #e5e9f0
label-empty-background = ${color.bg}

;; _-_-_-_-_-_-_-_-_-_-_-_-_-_-_-_-_-_-_-_-_-_

[module/name]
type = internal/xworkspaces

format = <label-state>
format-foreground = ${color.fg}
format-font = 3
format-padding = 0

label-active = "%name%"

label-occupied = 
label-urgent = 
label-empty = 

;; _-_-_-_-_-_-_-_-_-_-_-_-_-_-_-_-_-_-_-_-_-_

[module/title]
type = internal/xwindow

format = <label>
format-foreground = ${color.fg}
format-font = 2

label = %title%
label-maxlen = 20
label-empty = Desktop

;; _-_-_-_-_-_-_-_-_-_-_-_-_-_-_-_-_-_-_-_-_-_

[module/sep]
type = custom/text
content = " | "
content-font = 2

;;content-background = #000
content-foreground = ${color.fg}
;;content-padding = 4

[module/sep2]
type = custom/text
content = " "

;;content-background = #000
content-foreground = ${color.fg}
;;content-padding = 4

;; _-_-_-_-_-_-_-_-_-_-_-_-_-_-_-_-_-_-_-_-_-_
;;    __________  ______
;;   / ____/ __ \/ ____/
;;  / __/ / / / / /_    
;;  / /___/ /_/ / __/    
;;  /_____/\____/_/       
;;
;; _-_-_-_-_-_-_-_-_-_-_-_-_-_-_-_-_-_-_-_-_-_
```

## Scripts Externos

Los siguientes scripts personalizados son utilizados por Polybar para mostrar información dinámica. Deben guardarse en la ruta especificada (`/home/stark/.config/bspwm/scripts/`).

> [!tip] Personalización de Rutas
> Asegúrese de que la ruta `/home/stark/.config/bspwm/scripts/` en los archivos `.ini` y en las funciones Zsh coincida con la ubicación real de sus scripts. Si su nombre de usuario no es `stark`, deberá ajustarlo.

### `ethernet_status.sh`

Este script probablemente verifica el estado de la conexión Ethernet y muestra un icono o texto indicando si está activa o no.

[El código para Ethernet_status](https://pastebin.com/HcKxU3tD)

### `vpn_status.sh`

Este script monitorea el estado de la conexión VPN, mostrando si está activa y, posiblemente, la dirección IP asignada o el nombre del servidor.

[El código para vpn_status.sh](https://pastebin.com/sUk5hB4Q)

### `target_hack.sh`

Este script es crucial para entornos de laboratorio de hacking, ya que lee un archivo (`/home/stark/.config/bspwm/scripts/target`) para mostrar la dirección IP y el nombre de la máquina objetivo actual. Esto permite tener siempre visible el objetivo activo.

[El código para target_to_hack.sh](https://pastebin.com/Z0s6zy7W)

## Funciones Zsh para Gestión de Objetivos

Finalmente, se deben añadir las siguientes funciones a su archivo `~/.zshrc`. Estas funciones facilitan la configuración y limpieza del objetivo de hacking actual, interactuando directamente con el archivo que `target_hack.sh` lee.

> [!info] Integración con el Flujo de Trabajo
> Las funciones `settarget` y `cleartarget` son herramientas esenciales para un flujo de trabajo eficiente en laboratorios de hacking. Permiten al usuario definir rápidamente el objetivo actual (IP y nombre) y borrarlo cuando ya no es necesario, manteniendo la barra de Polybar actualizada.

```bash
function settarget(){
    ip_address=$1
    machine_name=$2
    echo "$ip_address $machine_name" > /home/stark/.config/bspwm/scripts/target
}

function cleartarget(){
    echo '' > /home/stark/.config/bspwm/scripts/target
}
```

Una vez configurado esto, se procederá con la instalación de `fzf`.